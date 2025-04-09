```C
/*
 * Macros to create function definitions for page flags
 */
#define FOLIO_TEST_FLAG(name, page)					\
static __always_inline bool folio_test_##name(const struct folio *folio) \
{ return test_bit(PG_##name, const_folio_flags(folio, page)); }

#define FOLIO_SET_FLAG(name, page)					\
static __always_inline void folio_set_##name(struct folio *folio)	\
{ set_bit(PG_##name, folio_flags(folio, page)); }

#define FOLIO_CLEAR_FLAG(name, page)					\
static __always_inline void folio_clear_##name(struct folio *folio)	\
{ clear_bit(PG_##name, folio_flags(folio, page)); }

#define __FOLIO_SET_FLAG(name, page)					\
static __always_inline void __folio_set_##name(struct folio *folio)	\
{ __set_bit(PG_##name, folio_flags(folio, page)); }

#define __FOLIO_CLEAR_FLAG(name, page)					\
static __always_inline void __folio_clear_##name(struct folio *folio)	\
{ __clear_bit(PG_##name, folio_flags(folio, page)); }

#define FOLIO_TEST_SET_FLAG(name, page)					\
static __always_inline bool folio_test_set_##name(struct folio *folio)	\
{ return test_and_set_bit(PG_##name, folio_flags(folio, page)); }

#define FOLIO_TEST_CLEAR_FLAG(name, page)				\
static __always_inline bool folio_test_clear_##name(struct folio *folio) \
{ return test_and_clear_bit(PG_##name, folio_flags(folio, page)); }

#define FOLIO_FLAG(name, page)						\
FOLIO_TEST_FLAG(name, page)						\
FOLIO_SET_FLAG(name, page)						\
FOLIO_CLEAR_FLAG(name, page)
```
