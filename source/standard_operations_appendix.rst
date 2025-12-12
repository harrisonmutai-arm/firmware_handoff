.. SPDX-License-Identifier: CC-BY-SA-4.0
.. SPDX-FileCopyrightText: Copyright The Firmware Handoff Specification Contributors

.. default-role:: code

.. _sec_operations:

Appendix A: Standard operations
===============================

This appendix describes the valid operations that may be performed on a TL in
more detail, in order to clarify how to use the various fields and to serve as a
guideline for implementation.

Validating a TL header
----------------------

.. default-role:: code

Inputs:

- `tl_base_addr`: Base address of the existing TL.

#. Compare `tl.signature` (`tl_base_addr + 0x0`) to `0x4a0f_b10b`. On a mismatch,
   abort (this is not a valid TL).

#. Compare `tl.version` (`tl_base_addr + 0x5`) to the expected version
   (currently |current_version|). If there is an exact match, the TL is valid
   for all operations outlined in this section. If `tl.version` is larger, the
   TL is valid for reading but must not be modified or relocated. If
   `tl.version` is smaller, either abort or switch to code designed to
   interpret the respective previous version of this specification (note that
   the version number `0x0` is illegal and processing should always abort if it
   is found).

#. *(optional)* Check that `tl.used_size` (`tl_base_addr + 0x8`) is smaller or equal
   to `tl.total_size` (`tl_base_addr + 0xc`), and that `tl.total_size` is smaller or
   equal to the size of the total area reserved for the TL (if known). If not,
   abort (TL is corrupted).

#. *(optional)* If `has_checksum`, check that the sum of `tl.size` bytes
   starting at `tl_base_addr` minus `tl.checksum` is equal to `tl.checksum`. If
   not, abort (TL is corrupted).

Reading a TL
------------

Inputs:

- `tl_base_addr`: Base address of the existing TL.

#. Calculate `te_base_addr` as `align8(tl_base_addr + tl.hdr_size)`. (Do not
   hardcode the value for `tl.hdr_size`!)

#. While `te_base_addr - tl_base_addr` is smaller or equal to `tl.used_size`:

   #. *(optional)* Check that `te_base_addr + te.hdr_size + te.data_size - tl_base_addr`
      is smaller or equal to `tl.used_size`, otherwise abort (the TL is corrupted).

   #. If `te.tag_id` (`te_base_addr + 0x0`) is a known tag, interpret the data
      at `te_base_addr + te.hdr_size` accordingly. (Do not hardcode the value
      for `te.hdr_size`, even for known tags!) Otherwise, ignore the tag and
      proceed with the next step.

   #. Add `align8(te.hdr_size + te.data_size)` to `te_base_addr`.

Adding a new TE
---------------

Inputs:

- `tl_base_addr`: Base address of the TL to add a TE to.
- `new_tag_id`: ID number of the tag for the new TE.
- `new_data_size`: Size in bytes of the data to be encapsulated in the TE.
- [data]: Data to be copied into the TE or generated on the fly.

#. *(optional)* Follow the steps in `Reading a TL`_ to look for a TE where
   `te.tag_id` is `0x0` (XFERLIST_VOID) and `te.data_size` is greater or equal
   to `new_data_size`. If found:

   #. Remember `te.data_size` as `old_void_data_size`.

   #. Use the `te_base_addr` of this tag for the rest of the operation.

   #. If `has_checksum`, Subtract the sum of `align8(new_data_size + 0x8)` bytes
      starting at `te_base_addr` from `tl.checksum`.

   #. Skip the next step (step 2) with all its substeps.

#. Calculate `te_base_addr` as `tl_base_addr + tl.used_size`.

   #. If `tl.total_size - tl.used_size` is smaller than `align8(new_data_size + 0x8)`,
      abort (not enough room to add TE).

   #. If `has_checksum`, subtract the sum of the 4 bytes from
      `tl_base_addr + 0x8` to `tl_base_addr + 0xc` from `tl.checksum`.

   #. Add `align8(new_data_size + 0x8)` to `tl.used_size`.

   #. If `has_checksum`, add the sum of the 4 bytes from `tl_base_addr + 0x8` to
      `tl_base_addr + 0xc` to `tl.checksum`.

#. Set `te.tag_id` (`te_base_addr + 0x0`) to `new_tag_id`.

#. Set `te.hdr_size` (`te_base_addr + 0x3`) to `8`.

#. Set `te.data_size` (`te_base_addr + 0x4`) to `new_data_size`.

#. Copy or generate the TE data into `te_base_addr + 0x8`.

#. If `has_checksum`, add the sum of `align8(new_data_size + 0x8)` bytes
   starting at `te_base_addr` to `tl.checksum`.

#. If an existing XFERLIST_VOID TE was chosen to be overwritten in step 1, and
   `old_void_data_size - new_data_size` is greater than or equal to `0x8`, then
   create a new void TE to fill the remaining space by calling `Adding a void TE`_
   with the following arguments:

   #. `void_te.base_addr` = `te_base_addr + align8(new_data_size + 0x8)`

   #. `void_te.size` =  `old_void_data_size - align8(new_data_size + 0x8)`

Removing a TE
-------------

Inputs:

- `te_base_addr`: Base address of the TE to be removed

#. Invoke `Adding a void TE`_ with following arguments

   #. `void_te.base_addr` = `te_base_addr`

   #. `void_te.size` = `te.data_size + te.hdr_size - 0x8`

Overwriting a TE
----------------

Inputs:

- `tl_base_addr`: Base address of the TL from which TE to be overwritten
- `te_base_addr`: Base address of the TE to be overwritten
- `new_data_size`: Size in bytes of the data to be encapsulated in the TE
- [data]: Data to be copied into the TE

#. If `te.data_size` is smaller than `new_data_size`, abort this operation here and instead first
   follow `Adding a void TE`_ with `te_base_addr` and `te.data_size`, then follow `Adding a new TE`_
   with `tl_base_addr`, `tag_id`, `new_data_size` and `[data]`

#. If `has_checksum`, xor the `te.data_size` bytes starting at `te_base_addr + te.hdr_size` with `tl.checksum`

#. Set `te.data_size` (`te_base_addr + 0x4`) to `align8(new_data_size)`

#. Copy or generate the new TE data into `te_base_addr + te.hdr_size`

#. If `has_checksum`, xor the `te.hdr_size + new_data_size` bytes starting at `te_base_addr` with `tl.checksum`

#. If `te.data_size - new_data_size` is greater or equal to `0x8` then call
   `Adding a void TE`_ with following arguments:

   #. `void.te.base_addr` = `te_base_addr + te.hdr_size + align8(new_data_size)`

   #. `void.te.size` =  `te.data_size - align8(new_data_size) - 0x8`

Adding a new TE with special data alignment requirement
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Inputs:

- `tl_base_addr`: Base address of the TL to add a TE to.
- `new_tag_id`: ID number of the tag for the new TE.
- `new_alignment`: The alignment boundary as a power of `2` that the data must be aligned to.
- `new_data_size`: Size in bytes of the data to be encapsulated in the TE.
- [data]: Data to be copied into the TE or generated on the fly.

#. Calculate `alignment_mask` as `(1 << new_alignment) - 1`.

#. If `(tl_base_addr + tl.used_size + 0x8) & alignment_mask` is not `0x0`, follow the
   steps in `Adding a new TE`_ with the following inputs (bypass the option to
   overwrite an existing XFERLIST_VOID TE):

   #. `tl_base_addr` remains the same

   #. `new_tag_id` is `0x0` (XFERLIST_VOID)

   #. `new_data_size` is `(1 << new_alignment) - ((tl_base_addr + tl.used_size + 0x8) & alignment_mask) - 0x8`.

   #. No data (i.e. just don't touch the bytes that form the data portion for this TE).

#. Follow the steps in `Adding a new TE`_ with the original inputs (again bypass
   the option to overwrite an existing XFERLIST_VOID TE).

#. If `new_alignment` is larger than `tl.alignment`:

   #. If `has_checksum`, subtract `tl.alignment` from `tl.checksum`.

   #. Set `tl.alignment` to `new_alignment`.

   #. If `has_checksum`, add `tl.alignment` to `tl.checksum`.

Creating a TL
^^^^^^^^^^^^^

Inputs:

- `tl_base_addr`: Base address where to place the new TL.
- `available_size`: Available size in bytes to reserve for the TL after `tl_base_addr`.

#. Check that `available_size` is larger than `0x18` (the assumed `tl.hdr_size`), otherwise abort.

#. Set `tl.signature` (`tl_base_addr + 0x0`) to `0x4a0f_b10b`.

#. Set `tl.checksum` (`tl_base_addr + 0x4`) to `0x0` (for now).

#. Set `tl.version` (`tl_base_addr + 0x5`) to |current_version|.

#. Set `tl.hdr_size` (`tl_base_addr + 0x6`) to `0x18`.

#. Set `tl.alignment` (`tl_base_addr + 0x7`) to `0x3`.

#. Set `tl.used_size` (`tl_base_addr + 0x8`) to `0x18` (the assumed `tl.hdr_size`).

#. Set `tl.total_size` (`tl_base_addr + 0xc`) to `available_size`.

#. If checksums are to be used, set `tl.flags` (`tl_base_addr + 0x10`) to `1`,
   else `0`. This is the value of `has_checksum`.

#. If `has_checksum`, calculate the checksum as the sum of all bytes from
   `tl_base_addr` to `tl_base_addr + tl.hdr_size`, and write the result to
   `tl.checksum`.

Relocating a TL
^^^^^^^^^^^^^^^

Inputs:

- `tl_base_addr`: Base address of the existing TL.
- `target_base`: Base address of the target region to relocate into.
- `target_size`: Size in bytes of the target region to relocate into.

#. Calculate `alignment_mask` as `(1 << tl.alignment) - 1`.

#. Calculate the current `alignment_offset` as `tl_base_addr & alignment_mask`.

#. Calculate `new_tl_base` as `(target_base & ~alignment_mask) + alignment_offset`.

#. If `new_tl_base` is below `target_base`, add `alignment_mask + 1` to `new_tl_base`.

#. If `new_tl_base - target_base + tl.used_size` is larger than `target_size`, abort
   (not enough space to relocate).

#. Copy `tl.used_size` bytes from `tl_base_addr` to `new_tl_base`.

#. If `has_checksum`, subtract the sum of the 4 bytes from `new_tl_base + 0xc`
   to `new_tl_base + 0x10` from `tl.checksum` (`new_tl_base + 0x4`).

#. Set `tl.total_size` (`new_tl_base + 0xc`) to `target_size - (new_tl_base - target_base)`.

#. If `has_checksum`, add the sum of the 4 bytes from `new_tl_base + 0xc` to
   `new_tl_base + 0x10` to `tl.checksum` (`new_tl_base + 0x4`).

.. note::
   After relocating a TL, implementations should consider scrubbing the old TL memory if it contains
   any secrets that might be accessible to later untrusted software.

Helper Routines
^^^^^^^^^^^^^^^

Adding a void TE
~~~~~~~~~~~~~~~~

Inputs:

- `te_base_addr`: Base address where void TE to be added
- `data_size`: Size in bytes of the data to be encapsulated in void TE

#. If `has_checksum`, subtract `te.hdr_size + data_size` bytes starting at `te_base_addr` from `tl.checksum`

#. Set `te.tag_id` (`te_base_addr + 0x0`) to `0x0` (XFERLIST_VOID)

#. Set `te.hdr_size` (`te_base_addr + 0x3`) to `0x8`

#. Set `te.data_size` (`te_base_addr + 0x4`) to `align8(data_size)`

#. *(optional)* Set the `data_size` bytes starting at `te_base_addr + te.hdr_size` to 0x0

#. If `has_checksum`, add `te.hdr_size + data_size` bytes starting at `te_base_addr` with `tl.checksum`



