---
name: extend-volume
description: >-
  Extend a Windows volume into unallocated space that another partition blocks,
  by relocating the blocker (usually the WinRE recovery partition) to the end of
  the disk. Use when "Extend Volume" is greyed out in Disk Management, when the
  user has unallocated space they cannot add to C:, when a "healthy partition"
  or recovery partition sits between C: and the free space, or after growing a
  VM disk / cloning to a larger drive leaves space stranded.
---

# Extend a volume past a blocking partition

Windows can only extend a partition into unallocated space that **immediately
follows it** on disk. Anything in between — nearly always the WinRE recovery
partition — greys out "Extend Volume", no matter how much free space exists.

The fix is not to move the blocker sideways. It is to **delete and recreate it at
the end of the disk**, which is the layout Microsoft ships anyway, and which stops
this recurring forever.

Steps 1–3 are strictly read-only. Do not delete, resize, or format anything before
the user approves in step 4.

Windows only. Requires an **elevated** session — verify first, because a
non-elevated `Get-Partition` succeeds while `Remove-Partition` fails halfway:

```powershell
$id = [Security.Principal.WindowsIdentity]::GetCurrent()
(New-Object Security.Principal.WindowsPrincipal($id)).IsInRole(
  [Security.Principal.WindowsBuiltInRole]::Administrator)
```

`$ARGUMENTS` may name the drive letter to extend. Default to `C`.

## 1. Survey the disk

Get the real layout. **Sort by `Offset`** — partition *numbers* do not reflect
physical order, and this whole procedure turns on physical order.

```powershell
Get-Disk | Format-Table Number,FriendlyName,
  @{n='Size(GB)';e={[math]::Round($_.Size/1GB,1)}},PartitionStyle,BusType -AutoSize

Get-Partition | Sort-Object DiskNumber,Offset | Format-Table DiskNumber,PartitionNumber,
  DriveLetter,@{n='Offset(GB)';e={[math]::Round($_.Offset/1GB,2)}},
  @{n='Size(GB)';e={[math]::Round($_.Size/1GB,2)}},Type,GptType,IsHidden -AutoSize

Get-Volume | Format-Table DriveLetter,FileSystemLabel,FileSystem,
  @{n='Size(GB)';e={[math]::Round($_.Size/1GB,2)}},
  @{n='Free(GB)';e={[math]::Round($_.SizeRemaining/1GB,2)}} -AutoSize
```

Confirm there is no headroom already — if `SizeMax` exceeds the current size, just
resize and skip this entire skill:

```powershell
Get-PartitionSupportedSize -DiskNumber 0 -PartitionNumber <target>
```

Unallocated space is not listed as a partition. Infer it from the gaps: compare
each `Offset + Size` against the next `Offset`, and the last one against `Disk.Size`.

## 2. Identify the blocker

Find the partition whose offset is **after** the target and **before** the
unallocated gap. Then branch on what it actually is — the `Type` and `GptType`
columns, not the label in Disk Management:

| Blocker | GPT type | Action |
| :------ | :------- | :----- |
| `Recovery` | `de94bba4-06d1-4d40-a16a-bfd50179d6ac` | This skill. Continue. |
| OEM / restore (Dell, Lenovo, HP) | vendor-specific | **Stop.** Deleting it destroys factory reset. Report and let the user decide. |
| Basic data with files | `ebd0a0a2-…` | **Stop.** It holds user data. Offer to move the data off first. |
| `System` (EFI) / `Reserved` (MSR) | — | Never touch. If one of these is the blocker the layout is unusual; report and stop. |

If the gap sits *before* the target partition, no relocation of the blocker helps —
Windows cannot extend backwards. Say so and stop; that needs third-party tooling
that moves the target itself.

## 3. Check WinRE health

```powershell
reagentc /info
```

Record the current **status** and **WinRE version** — you need them to prove
success later. If status is already `Disabled`, WinRE may have been broken before
you arrived; say so rather than silently "fixing" it.

Also check BitLocker on the target, since resizing an encrypted volume is a
different conversation:

```powershell
Get-BitLockerVolume -MountPoint "C:" | Format-List MountPoint,ProtectionStatus,VolumeStatus
```

If protection is On, have the user suspend it (`Suspend-BitLocker -MountPoint "C:"
-RebootCount 1`) before proceeding.

## 4. Present the plan and get approval

Show the user the current layout, name the blocker, and state the resulting size.
Then **require a backup or snapshot before continuing.** This step rewrites the
partition table; a power loss mid-resize can leave the volume inconsistent. On a
VM this is a snapshot and costs seconds — insist on it.

Do not proceed until they confirm. Offer the zero-risk alternative too: leave the
space as a separate volume. It touches nothing, and for some users it is enough.

## 5. Stage WinRE onto the target volume

`reagentc /disable` copies `winre.wim` out of the recovery partition and back into
`C:\Windows\System32\Recovery`, which is what makes deleting the partition safe.

```powershell
reagentc /disable
```

**Verify the image actually landed before deleting anything.** If this file is
missing, `reagentc /enable` will have nothing to restore and the user loses their
recovery environment permanently:

```powershell
reagentc /info                                    # expect: status Disabled
Get-Item 'C:\Windows\System32\Recovery\winre.wim' -Force |
  Format-List Name,@{n='Size(MB)';e={[math]::Round($_.Length/1MB,1)}},LastWriteTime
```

Expect roughly 400–800 MB. **If it is absent, stop here** — re-enable WinRE and
report. Do not delete the partition to "see if it works".

## 6. Delete, extend, recreate

Delete the blocker, then extend the target into the merged free space, holding
back 1 GB at the end for the new recovery partition:

```powershell
Remove-Partition -DiskNumber 0 -PartitionNumber <blocker> -Confirm:$false

$s = Get-PartitionSupportedSize -DiskNumber 0 -PartitionNumber <target>
Resize-Partition -DiskNumber 0 -PartitionNumber <target> -Size ($s.SizeMax - 1GB)
```

Create the replacement in the trailing gap, with the recovery type GUID:

```powershell
$p = New-Partition -DiskNumber 0 -UseMaximumSize `
  -GptType '{de94bba4-06d1-4d40-a16a-bfd50179d6ac}'
Format-Volume -Partition $p -FileSystem NTFS -NewFileSystemLabel 'Recovery' -Confirm:$false
```

Set the GPT attributes. PowerShell cannot do this — `diskpart` is required. The
value is `GPT_ATTRIBUTE_PLATFORM_REQUIRED` (`0x1`) plus `NO_DRIVE_LETTER`
(`0x8000000000000000`); without it Windows may assign the partition a letter and
expose it in Explorer:

```powershell
@'
select disk 0
select partition <new>
gpt attributes=0x8000000000000001
'@ | diskpart
```

On **MBR** disks instead of GPT, there is no type GUID or attribute field — use
`set id=27` in `diskpart` to mark it as the hidden recovery type. Note the 4
primary partition limit.

## 7. Re-enable and verify

```powershell
reagentc /enable
reagentc /info
```

**"Operation Successful" from `/enable` is not proof.** Check all three:

- **Status** is `Enabled`.
- **Location** is a real path (`…\harddisk0\partition4\Recovery\WindowsRE`), not empty.
- **WinRE version** matches what step 3 recorded — *not* `0.0.0.0`.

A failed registration reports `Enabled` with a zeroed version and empty location.
That is the classic silent failure here, and it is the single check worth caring
about. If it fails, WinRE still works from `C:\Windows\System32\Recovery` — the
user is not stranded — but re-registration needs fixing before they reboot.

Confirm the final layout with the step 1 commands. Then report: the new size, the
free space gained, and the three WinRE facts above.

**Tell the user to reboot and confirm Windows starts before discarding the
snapshot.** Everything verifies from inside the running OS, but the EFI/BCD path
is only truly proven by an actual boot. `reagentc /boottore` then a restart boots
into WinRE once, if they want to verify recovery itself.

## Notes

- Sizing: Microsoft's reference layout uses ~1 GB for recovery. Older machines
  ship 300–600 MB, which is why WinRE servicing updates (the KB5034441 `0x80070643`
  failure) fail on them. Recreating at 1 GB fixes that as a side effect — worth
  mentioning to the user, as it is a problem they may not have hit yet.
- Windows Update and feature upgrades can create a *second* recovery partition
  rather than grow the first. If the survey shows several, the extras are usually
  orphans — but confirm against `reagentc /info`, which names the live one, before
  calling any of them dead.
- `reagentc /disable` leaves the old partition intact but unreferenced. If the user
  aborts between steps 5 and 6, `reagentc /enable` restores the original state.
- Never `Remove-Partition` the EFI System or MSR partition. The machine will not boot.
- Dynamic disks and Storage Spaces do not work this way; this skill is for basic
  disks. `Get-Disk` reporting anything other than a basic disk means stop.
- If `Resize-Partition` fails with a shrink/extend error despite free space, an
  immovable file (page file, hibernation, shadow copies) may sit at the boundary.
  That blocks *shrinking*, not the extending this skill does — if it appears here,
  the layout is not what the survey suggested. Re-check before forcing anything.
