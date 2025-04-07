```C
/**
 * folio_attach_private - Attach private data to a folio.
 * @folio: Folio to attach data to.
 * @data: Data to attach to folio.
 *
 * Attaching private data to a folio increments the page's reference count.
 * The data must be detached before the folio will be freed.
 */
static inline void folio_attach_private(struct folio *folio, void *data)
{
	folio_get(folio);/*先增加引用计数*/
	folio->private = data;/*然后改变指针指向*/
	folio_set_private(folio);/**/
}
/**
 * folio_change_private - Change private data on a folio.
 * @folio: Folio to change the data on.
 * @data: Data to set on the folio.
 *
 * Change the private data attached to a folio and return the old
 * data.  The page must previously have had data attached and the data
 * must be detached before the folio will be freed.
 *
 * Return: Data that was previously attached to the folio.
 */
static inline void *folio_change_private(struct folio *folio, void *data)
{
	void *old = folio_get_private(folio);

	folio->private = data;
	return old;
}

/**
 * folio_detach_private - Detach private data from a folio.
 * @folio: Folio to detach data from.
 *
 * Removes the data that was previously attached to the folio and decrements
 * the refcount on the page.
 *
 * Return: Data that was attached to the folio.
 */
static inline void *folio_detach_private(struct folio *folio)
{
	void *data = folio_get_private(folio);

	if (!folio_test_private(folio))
		return NULL;
	folio_clear_private(folio);
	folio->private = NULL;
	folio_put(folio);

	return data;
}
```