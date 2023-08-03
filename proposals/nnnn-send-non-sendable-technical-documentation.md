# Send Non-Sendable Pass, implementation documentation

`SendNonSendable` is implemented as a raw SIL pass. The SIL pass performs a flow-sensitive, intraprocedural analysis for each function body that assigns a partition to the non-sendable values in scope at each point in its body. This partition groups the values into regions, with some values grouped into the special "transferred" region indicating that their region was transferred by an isolation-crossing application and can no longer be accessed. Only values that have been initialized by a given program point are tracked in the partition for that program point. Each SIL instruction is mapped to an operation on these partitions that manipulates the set of tracked values and their assignments to regions. Each operation takes arguments, whose values are not directly SILValues but rather a parallel set of values referred to in the code as `TrackableSILValue`s. The `SendNonSendable` pass maps each SILValue to a `TrackableSILValue`, often mapping two SILValues to the same `TrackableSILValue` if one is a projection of the other or refers to the same underlying storage. To indicate this layer of indirection `%%` (double percentage signs) are used to indicate `TrackableSILValue`s.

There are 5 operations on partitions that source SILInstructions get translated to:

- `Assign %%0 = %%1`
  Before this operation, `%%1` must already be tracked by the partition, and assigned to a non-transferred region. `%%0` need not be tracked or non-transferred if it is tracked. After this operation, `%%1`'s assignment in the partition will not change, and `%%0` will be tracked and assigned to the same region as %%1.

- `AssignFresh %%0`
  Before this operation, `%%0` need not be tracked or non-transferred if it is tracked. After this operation, `%%0` will be assigned to a region that no other values in the partition are assigned to.

- `Transfer %%0`
  Before this operation, `%%0` must be tracked by the partition but may or may not be transferred. After this operation, `%%0` will be marked as transferred in the partition. If `%%0` was assigned to a non-transferred region before the operation, then all other values in that region will also be marked as transferred after the operation.

- `Merge %%0 with %%1`
  Before this operation, both `%%0` and `%%1` must be tracked and assigned to non-transferred regions. After this operation, all values in both of those regions will be assigned to a single, non-transferred region.

- `Require %%0`
  Before this operation, `%%0` must be tracked and assigned to a non-transferred region. This operation has no effect on the partition.

TODO: add examples of SILInstructions that translate to each PartititonOp

At entry to a function body, `self` (if non-sendable) and all non-sendable arguments are assigned to a single, non-transferred region. Using fixpoint iteration, partitions are then computed at entry and exit to each basic block of the function that satisfy two properties:

1. Applying all operations of a basic block, in order, to its entry partition yields its exit partition
2. Each entry partition is the "join" of the exit partitions of each predecessor to its basic basic block. The join of a set of partitions is the finest partition in which any two values that are assigned to the same region in some partition in the set are assigned to the same region, and all values that are transferred in some partition in the set are transferred.

Once these partitions are computed, the preconditions of each operations can be checked against them, and diagnostics are reported if they do not hold.