.. SPDX-License-Identifier: CC-BY-SA-4.0
.. SPDX-FileCopyrightText: Copyright The Firmware Handoff Specification Contributors

.. default-role:: code

.. _sec_operations:

Appendix A: Standard operations
===============================

This appendix describes the valid operations that may be performed on a TL in
more detail, in order to clarify how to use the various fields and to serve as a
guideline for implementation.

.. _op_validate_tl:

Validating a TL header
----------------------

.. function:: validate_tl_header(tl_base_addr)

   :param tl_base_addr: Base address of the existing TL.
   :type tl_base_addr: int

   #. Compare `tl.signature` (`tl_base_addr + 0x0`) to `0x4a0f_b10b`. On a mismatch,
      abort (this is not a valid TL).
   #. Compare `tl.version` (`tl_base_addr + 0x5`) to the expected version
      (currently |current_version|). If there is an exact match, the TL is valid
      for all operations outlined in this appendix. If `tl.version` is larger, the
      TL is valid for reading but must not be modified or relocated. If
      `tl.version` is smaller, either abort or switch to code designed to
      interpret the respective previous version of this specification (note that
      the version number `0x0` is illegal and processing should always abort if it
      is found).
   #. *(Optional)* Confirm that `tl.used_size` (`tl_base_addr + 0x8`) is smaller
      or equal to `tl.total_size` (`tl_base_addr + 0xc`), and that `tl.total_size`
      fits within the reserved TL area (if known). Otherwise, abort because the TL
      is corrupted.
   #. *(Optional)* If `has_checksum`, verify that the sum of `tl.size` bytes
      starting at `tl_base_addr`, minus `tl.checksum`, equals `tl.checksum`.
      Abort on mismatch as this indicates corruption.

.. _op_read_tl:

Reading a TL
------------

.. function:: read_tl(tl_base_addr)

   :param tl_base_addr: Base address of the existing TL.
   :type tl_base_addr: int

   #. Calculate `te_base_addr` as `align8(tl_base_addr + tl.hdr_size)` (do not
      hardcode the value of `tl.hdr_size`).
   #. While `te_base_addr - tl_base_addr` is smaller or equal to `tl.used_size`:

      #. *(Optional)* Check that
         `te_base_addr + te.hdr_size + te.data_size - tl_base_addr` is smaller or
         equal to `tl.used_size`, otherwise abort (the TL is corrupted).
      #. If `te.tag_id` (`te_base_addr + 0x0`) is a known tag, interpret the data
         at `te_base_addr + te.hdr_size` accordingly. (Do not hardcode the value
         for `te.hdr_size`, even for known tags.) Otherwise, ignore the tag and
         proceed with the next step.
      #. Add `align8(te.hdr_size + te.data_size)` to `te_base_addr`.

.. _op_add_te:

Adding a new TE
---------------

.. function:: add_transfer_entry(tl_base_addr, new_tag_id, new_data_size, data=None)

   :param tl_base_addr: Base address of the TL that is being extended.
   :type tl_base_addr: int
   :param new_tag_id: Identifier for the new TE.
   :type new_tag_id: int
   :param new_data_size: Data payload size in bytes.
   :type new_data_size: int
   :param data: Optional payload to copy into the TE.
   :type data: bytes

   #. *(Optional pre-check)* Follow :ref:`op_read_tl` to locate a TE where
      `te.tag_id` is `0x0` (XFERLIST_VOID) and `te.data_size` is greater or equal
      to `new_data_size`. If a suitable void TE is found:

      #. Remember `te.data_size` as `old_void_data_size`.
      #. Record the entry base address as `void_te_base_addr` and reuse it as
         `te_base_addr` for the remainder of the procedure.
      #. If `has_checksum`, subtract the sum of `align8(new_data_size + 0x8)` bytes
         starting at `te_base_addr` from `tl.checksum`.
      #. Skip step 2 and continue with step 3.

   #. Calculate `te_base_addr` as `tl_base_addr + tl.used_size`.

      #. If `tl.total_size - tl.used_size` is smaller than
         `align8(new_data_size + 0x8)`, abort (not enough space to add the TE).
      #. If `has_checksum`, subtract the sum of the 4 bytes from
         `tl_base_addr + 0x8` to `tl_base_addr + 0xc` from `tl.checksum`.
      #. Add `align8(new_data_size + 0x8)` to `tl.used_size`.
      #. If `has_checksum`, add the sum of the 4 bytes from `tl_base_addr + 0x8` to
         `tl_base_addr + 0xc` to `tl.checksum`.

   #. Program the TE:

      #. Set `te.tag_id` (`te_base_addr + 0x0`) to `new_tag_id`.
      #. Set `te.hdr_size` (`te_base_addr + 0x3`) to `0x8`.
      #. Set `te.data_size` (`te_base_addr + 0x4`) to `new_data_size`.
      #. Copy or generate `data` into `te_base_addr + 0x8`. If `data` is omitted,
         populate the region according to the TE definition.
      #. If `has_checksum`, add the sum of `align8(new_data_size + 0x8)` bytes
         starting at `te_base_addr` to `tl.checksum`.

   #. If an existing XFERLIST_VOID TE was repurposed in step 1 and
      `old_void_data_size - new_data_size` is greater or equal to `0x8`, create a
      replacement void entry by calling :ref:`op_add_void` with:

      #. `te_base_addr = void_te_base_addr + align8(new_data_size + 0x8)`.
      #. `void_data_size = old_void_data_size - align8(new_data_size + 0x8)`.

.. _op_remove_te:

Removing a TE
-------------

.. function:: remove_transfer_entry(tl_base_addr, te_base_addr)

   :param tl_base_addr: Base address of the TL that contains the TE.
   :type tl_base_addr: int
   :param te_base_addr: Base address of the TE to be removed.
   :type te_base_addr: int

   #. Read `te.hdr_size` and `te.data_size` from the TE header.
   #. Compute `void_data_size` as `align8(te.hdr_size + te.data_size) - 0x8`.
   #. Call :ref:`op_add_void` with `te_base_addr` and `void_data_size` to convert
      the TE into an XFERLIST_VOID entry of equivalent size.

.. _op_overwrite_te:

Overwriting a TE
----------------

.. function:: overwrite_transfer_entry(tl_base_addr, te_base_addr, new_data_size, data, new_tag_id=None)

   :param tl_base_addr: Base address of the TL that contains the TE.
   :type tl_base_addr: int
   :param te_base_addr: Base address of the TE to be updated.
   :type te_base_addr: int
   :param new_data_size: New payload size in bytes.
   :type new_data_size: int
   :param data: Replacement payload.
   :type data: bytes
   :param new_tag_id: Optional replacement tag ID (defaults to the existing tag).
   :type new_tag_id: int

   #. If `new_data_size` is larger than the existing `te.data_size`, first call
      :ref:`op_remove_te` on `te_base_addr` and then invoke :ref:`op_add_te`
      with the desired parameters (using either the original or updated tag ID).
   #. Otherwise, proceed in place:

      #. Store the current `te.data_size` as `old_data_size`.
      #. If `has_checksum`, subtract the sum of `align8(old_data_size + te.hdr_size)`
         bytes starting at `te_base_addr` from `tl.checksum`.
      #. If `new_tag_id` is provided, update `te.tag_id` accordingly.
      #. Set `te.data_size` (`te_base_addr + 0x4`) to `new_data_size`.
      #. Copy or generate the new payload into `te_base_addr + te.hdr_size`.
      #. If `has_checksum`, add the sum of `align8(new_data_size + te.hdr_size)`
         bytes starting at `te_base_addr` to `tl.checksum`.
      #. If `old_data_size - new_data_size` is greater or equal to `0x8`, create a
         trailing void entry by calling :ref:`op_add_void` with:

         #. `te_base_addr = te_base_addr + te.hdr_size + align8(new_data_size)`.
         #. `void_data_size = align8(old_data_size) - align8(new_data_size) - 0x8`.

.. _op_add_aligned_te:

Adding a new TE with special alignment requirements
---------------------------------------------------

.. function:: add_transfer_entry_aligned(tl_base_addr, new_tag_id, new_alignment, new_data_size, data=None)

   :param tl_base_addr: Base address of the TL to add a TE to.
   :type tl_base_addr: int
   :param new_tag_id: Identifier for the new TE.
   :type new_tag_id: int
   :param new_alignment: Alignment requirement expressed as a power of two.
   :type new_alignment: int
   :param new_data_size: Data payload size in bytes.
   :type new_data_size: int
   :param data: Optional payload to copy into the TE.
   :type data: bytes

   #. Calculate `alignment_mask` as `(1 << new_alignment) - 1`.
   #. If `(tl_base_addr + tl.used_size + 0x8) & alignment_mask` is not `0x0`,
      insert padding by invoking :ref:`op_add_te` with:

      #. `new_tag_id = 0x0` (XFERLIST_VOID).
      #. `new_data_size = (1 << new_alignment) - ((tl_base_addr + tl.used_size + 0x8) & alignment_mask) - 0x8`.
      #. No payload (`data=None`).

   #. Call :ref:`op_add_te` with the original inputs (bypassing the option to
      overwrite an existing XFERLIST_VOID TE).
   #. If `new_alignment` is larger than `tl.alignment`:

      #. If `has_checksum`, subtract the old `tl.alignment` from `tl.checksum`.
      #. Set `tl.alignment` to `new_alignment`.
      #. If `has_checksum`, add the new `tl.alignment` to `tl.checksum`.

.. _op_create_tl:

Creating a TL
-------------

.. function:: create_transfer_list(tl_base_addr, available_size, has_checksum=False)

   :param tl_base_addr: Base address where the TL is instantiated.
   :type tl_base_addr: int
   :param available_size: Bytes reserved for the TL after `tl_base_addr`.
   :type available_size: int
   :param has_checksum: Whether checksum protection is enabled.
   :type has_checksum: bool

   #. Check that `available_size` is larger than `0x18` (the assumed `tl.hdr_size`);
      otherwise abort.
   #. Set `tl.signature` (`tl_base_addr + 0x0`) to `0x4a0f_b10b`.
   #. Set `tl.checksum` (`tl_base_addr + 0x4`) to `0x0`.
   #. Set `tl.version` (`tl_base_addr + 0x5`) to |current_version|.
   #. Set `tl.hdr_size` (`tl_base_addr + 0x6`) to `0x18`.
   #. Set `tl.alignment` (`tl_base_addr + 0x7`) to `0x3`.
   #. Set `tl.used_size` (`tl_base_addr + 0x8`) to `0x18`.
   #. Set `tl.total_size` (`tl_base_addr + 0xc`) to `available_size`.
   #. If `has_checksum`, set `tl.flags` (`tl_base_addr + 0x10`) to `1`; otherwise
      set it to `0`.
   #. If `has_checksum`, calculate the checksum as the sum of all bytes from
      `tl_base_addr` to `tl_base_addr + tl.hdr_size`, and store the result in
      `tl.checksum`.

.. _op_relocate_tl:

Relocating a TL
---------------

.. function:: relocate_transfer_list(tl_base_addr, target_base, target_size)

   :param tl_base_addr: Base address of the TL being relocated.
   :type tl_base_addr: int
   :param target_base: Base address of the destination buffer.
   :type target_base: int
   :param target_size: Size in bytes of the destination buffer.
   :type target_size: int

   #. Calculate `alignment_mask` as `(1 << tl.alignment) - 1`.
   #. Calculate the current `alignment_offset` as `tl_base_addr & alignment_mask`.
   #. Calculate `new_tl_base` as `(target_base & ~alignment_mask) + alignment_offset`.
   #. If `new_tl_base` is below `target_base`, add `alignment_mask + 1` to `new_tl_base`.
   #. If `new_tl_base - target_base + tl.used_size` is larger than `target_size`,
      abort (not enough space to relocate).
   #. Copy `tl.used_size` bytes from `tl_base_addr` to `new_tl_base`.
   #. If `has_checksum`, subtract the sum of the 4 bytes from `new_tl_base + 0xc`
      to `new_tl_base + 0x10` from `tl.checksum` (`new_tl_base + 0x4`).
   #. Set `tl.total_size` (`new_tl_base + 0xc`) to
      `target_size - (new_tl_base - target_base)`.
   #. If `has_checksum`, add the sum of the 4 bytes from `new_tl_base + 0xc` to
      `new_tl_base + 0x10` to `tl.checksum` (`new_tl_base + 0x4`).

.. note::
   After relocating a TL, implementations should consider scrubbing the original TL
   memory if it contained secrets that could be exposed to untrusted software.

Helper routines
---------------

.. _op_add_void:

Adding a void TE
~~~~~~~~~~~~~~~~

.. function:: add_void_transfer_entry(tl_base_addr, te_base_addr, void_data_size)

   :param tl_base_addr: Base address of the TL that contains the void entry.
   :type tl_base_addr: int
   :param te_base_addr: Base address where the void TE resides.
   :type te_base_addr: int
   :param void_data_size: Size of the void payload in bytes (must be a multiple of 8).
   :type void_data_size: int

   #. Let `void_entry_size` be `align8(void_data_size) + 0x8`.
   #. If `has_checksum`, subtract the sum of the current `void_entry_size` bytes
      at `te_base_addr` from `tl.checksum`.
   #. Set `te.tag_id` (`te_base_addr + 0x0`) to `0x0` (XFERLIST_VOID).
   #. Set `te.hdr_size` (`te_base_addr + 0x3`) to `0x8`.
   #. Set `te.data_size` (`te_base_addr + 0x4`) to `align8(void_data_size)`.
   #. *(Optional)* Zero the `void_data_size` bytes starting at
      `te_base_addr + te.hdr_size`.
   #. If `has_checksum`, add the sum of the updated `void_entry_size` bytes at
      `te_base_addr` to `tl.checksum`.

