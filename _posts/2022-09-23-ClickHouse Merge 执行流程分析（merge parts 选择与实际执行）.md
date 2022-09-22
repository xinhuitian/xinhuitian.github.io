---
layout: post
title:  "ClickHouse Merge 执行流程分析（merge parts 选择与实际执行）"
date:   2022-09-23 00:08 +0800
categories: ClickHouse
---
基于 ClickHouse 21.11.5 版本

# Parts 选择：selectPartsToMerge

scheduleDataProcessingJob 每次调用时都会首先调用 selectPartsToMerge 确定是否有需要被 merge 的 parts，如果有，会生成一个 MergeMutateSelectedEntry 类型的 merge_entry。

首先计算 max_source_parts_size， 一般为 max_bytes_to_merge_at_max_space_in_pool，默认为 150ULL * 1024 * 1024 * 1024， 也就是 150 GB，如果这个值计算出来不为 0，则开始选取 partsToMerge。如果这个值设置为 0， 应该就等同于关闭 merge 功能了。

实际的 selectPartsToMerge 功能的实现主要在 MergeTreeDataMergerMutator::selectPartsToMerge 这个方法中。首先比较粗略地找到所有可以 merge 的 part ranges。

## Parts Ranges 生成

首先找到第一个可以 merge 的 data part，判断条件为没有正在参加某个 merge 或 mutate task 就可以。找到以后，标记为 prev_part, 添加到 parts_ranges 数组的最末一个 vector 中，之后继续寻找可以一起 merge 的 data parts。后续寻找的逻辑除了没有参加正在进行的 merge 或 mutate 以外，还需要判断与 prev_part 的人 mutation version 是否一致，以及是否包含了相同的 projections。全部条件满足，则将其加入到 prev_part 所在的 parts_range 中。如果不满足，则上一个 parts_range 计算结束，创建下一个 parts_range 继续进行以上的操作。

## Parts Range Select

拿到所有可以 merge 的 Parts Range 以后，需要选择出一个 Parts Range 来进行本轮的 Merge。

如果定义了 TTL，会首先使用 delete_ttl_selector 进行过滤。默认情况，不会走这个逻辑。

如果 TTL 没有进行选择，则使用 SimpleMergeSelector 进行选择。

对于每个 Part Range，SimpleMergeSelector 还会遍历其每个大于 1 个 part 的子区间，基于每个子区间的 range size, sum size, max size, part number 等信息计算出一个数值，并与一个 lower_base 进行比较，如果大于 lower_base，则这个子区间可以被 merge。

每当遇到一个可以 merge 的 sub part range 时，SimpleMergeSelector 会通过一个 Estimator 计算一个分数出来，并合当前最高分进行比较，如果更少，则选取这个区间作为 best range。等所有 sub part range 都被遍历过以后，最终的 best range 会作为 selected parts range 返回。

## Merge Select 算法

对每个 range 来说：

* size_normalized：0 到 1 之间。range part sum size 越大，越趋近于 1
* age_normalized ：0 到 1 之间。range part 的 min_age 越大，越趋近于 1。min_age 有一个范围，最小为 10。最大值和 sum_size 相关，sum_size 越小， 最大值越接近于 24 小时。sum_size 越大，最大值越接近于 30 天。
* num_parts_normalized：0 到 1 之间。默认控制 part size 在 10 到 150 之间，将这之间的 part number 归一化处理。
* combined_ratio: 1.0 与 age_normalized + num_parts_normalized 中取更小值，也是一个 0 到 1 的值
* lowered_base: base 的默认值为 5， 取 2 到 5 中间的一个值，combined_ratio 越小，值越大。
    * age 越大，num_parts 越大，lowered_base 越小

允许 merge 的条件：

* range_size * (avg_size + cost_to_add) / (max_size + cost_to_add) ≥ lowered_base
* 简单来说：part size 的差距越小，age 越大，num_parts 越大，越容易被 merge


是否最终一定会 merge 成一个 part？

* 按照默认值来说，lowered_based 最小会取到 2.0， 如果仅剩下 2 个 data parts，同时其大小相差过大的话，最终应该就不会被 merge。不太确定这种情况是否存在。

# Merge 执行：MergePlainMergeTreeTask 

当有 parts 被选择 merge 时，会创建一个 MergePlainMergeTreeTask 提交给 background assignee 去调度执行。MergePlainMergeTreeTask 在执行时，会不断调用 executeStep 执行不同阶段的任务。

MergePlainMergeTreeTask 创建时 state 会设置为 State::NEED_PREPARE。 在第一次 executeStep 时，会调用到 prepare() 方法，调用 MergeTreeDataMergerMutator::mergePartsToTemporaryPart 来创建一个 MergeTask 并执行。调用完成后，设置 state 为 State::NEED_EXECUTE，executeStep 返回 true，意味着这个 task 需要被 execute again。

下一次执行的时候，就会调用 prepare 阶段创建的 MergeTask 的 execute 方法，执行真正的 merge。执行完成后，会将 state 设置为 state::NEED_FINISH， 再次返回 true。下一次被调用时，调用 finish 方法， 最终调用 MergeTreeDataMergerMutator::renameMergedTemporaryPart 进行最后的 data part 处理，并将 state 设置为 state::SUCCESS, 返回 false，executeStep 不会继续循环执行，task 成功结束。
