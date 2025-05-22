```C
/**
 * __is_extent_mergeable - Check if two adjacent extents can be merged.
 * @back: The preceding extent info.
 * @front: The succeeding extent info.
 * @type: Extent type.
 *
 * Determines if 'back' and 'front' represent a single contiguous range,
 * both logically (file offsets) and physically (block addresses for EX_READ)
 * or based on age similarity (for EX_BLOCK_AGE).
 * Includes specific checks for compressed extents.
 *
 * Returns: true if mergeable, false otherwise.
 */
static bool __is_extent_mergeable(struct extent_info *back,
		struct extent_info *front, enum extent_type type)
{
	if (type == EX_READ) { // For read extents
#ifdef CONFIG_F2FS_FS_COMPRESSION
		// Compression specific check:
		// An extent representing a compressed range has c_len > 0.
		// If c_len is set BUT the extent's length (len) doesn't match c_len,
		// it means this extent represents only *part* of a contiguous compressed
		// cluster group (this shouldn't normally happen with how extents are added,
		// but this check prevents merging such partial representations).
		// We only merge extents that either aren't compressed (c_len=0) or
		// represent a complete contiguous compressed cluster group (len == c_len).
		if (back->c_len && back->len != back->c_len)
			return false;
		if (front->c_len && front->len != front->c_len)
			return false;
		// Note: This implies merging two compressed extents requires both to represent
		// full cluster groups, and they must be contiguous both in file offset and block address.
		// The resulting merged extent might have its c_len reset by __set_extent_info unless
		// the merge logic is extended to calculate a combined c_len. Currently, it seems
		// a merge of two c_len>0 extents would result in a c_len=0 extent unless handled specially.
		// However, the primary use case here seems to be merging identical single-cluster extents.

#endif
		// Standard check: File offsets must be contiguous AND block addresses must be contiguous.
		return (back->fofs + back->len == front->fofs &&
				back->blk + back->len == front->blk);
	} else if (type == EX_BLOCK_AGE) { // For block age extents (GC)
		// Check: File offsets must be contiguous AND age/last_blocks values must be similar enough.
		return (back->fofs + back->len == front->fofs &&
			abs(back->age - front->age) <= SAME_AGE_REGION &&
			abs(back->last_blocks - front->last_blocks) <=
							SAME_AGE_REGION);
	}
	// Unknown type
	return false;
}
```