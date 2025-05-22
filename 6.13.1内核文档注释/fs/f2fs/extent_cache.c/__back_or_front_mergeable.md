```C
/**
 * __is_back_mergeable - Check if 'cur' extent can be merged following 'back'.
 * @cur: The new extent info.
 * @back: The preceding extent info.
 * @type: Extent type.
 *
 * Wrapper for __is_extent_mergeable.
 */
static bool __is_back_mergeable(struct extent_info *cur,
		struct extent_info *back, enum extent_type type)
{
	// Check if 'back' followed immediately by 'cur' forms a contiguous extent
	return __is_extent_mergeable(back, cur, type);
}

/**
 * __is_front_mergeable - Check if 'front' extent can be merged following 'cur'.
 * @cur: The new extent info.
 * @front: The succeeding extent info.
 * @type: Extent type.
 *
 * Wrapper for __is_extent_mergeable.
 */
static bool __is_front_mergeable(struct extent_info *cur,
		struct extent_info *front, enum extent_type type)
{
	// Check if 'cur' followed immediately by 'front' forms a contiguous extent
	return __is_extent_mergeable(cur, front, type);
}
```