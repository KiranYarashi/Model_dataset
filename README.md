# Core Suite Business Logic Developer Training - Hands-on Guide

This document provides step-by-step instructions for adding a new "Group Notes" feature.

## Step 1: Create of a new Database table `M_GROUP_NOTES`

**Action:** This step is performed directly in your database management tool (e.g., SQL Server Management Studio).

**Hints:**
*   Ensure data types match the specification (e.g., `INT`, `DATETIME`, `VARCHAR`, `SMALLINT`).
*   Define `pk` as `IDENTITY` and `NOT NULL`.
*   Set `NOT NULL` constraints where specified.
*   Set `DEFAULT ((0))` for `discriminator`.

## Step 2: Record_Set (`ARSMGroupNotes.h` & `ARSMGroupNotes.cpp`)

### File: `ARSMGroupNotes.h`

*   **Class Definition:**
    *   Declare class `ARSMGroupNotes` inheriting from `AlterableRecordset`.
    *   Use `DECLARE_DYNAMIC(ARSMGroupNotes)`.
*   **Member Variables (within `//{{AFX_FIELD...` block):**
    *   `m_pk`, `m_group_id`, `m_opening_reg_date` (as `ATime`), `m_opening_effec_date` (as `ATime`), `m_opening_status`, `m_opening_ref` (as `CString`), etc., for all table columns.
*   **Virtual Function Overrides (within `//{{AFX_VIRTUAL...` block):**
    *   `GetDefaultConnect()`
    *   `GetDefaultSQL()`
    *   `DoFieldExchange(CFieldExchange* pFX)`
*   **Getter/Setter Declarations:**
    *   For each member variable: `GetFieldName()`, `SetFieldName()`, `IsNullFieldName()`.
    *   Example: `long GetPk(); void SetPk(long lValue); bool IsNullPk();`

### File: `ARSMGroupNotes.cpp`

*   **Include:** `"stdafx.h"`, `"ARSMGroupNotes.h"`.
*   **MFC Macro:** `IMPLEMENT_DYNAMIC(ARSMGroupNotes, AlterableRecordset)`.
*   **Constructor `ARSMGroupNotes(ADatabase* pdb)`:**
    *   Initialize member variables in `//{{AFX_FIELD_INIT...` block (e.g., `m_pk = 0;`, `m_opening_ref = _T("");`).
    *   Set `m_nFields` to the total number of columns.
    *   Set `m_nDefaultType` (e.g., `snapshot`).
    *   Set `m_strTable = _T("M_GROUP_NOTES");`.
    *   Call `AddColumn()` for each field:
        *   `AddColumn(_T("pk"), &m_pk, SQL_INTEGER, true, true);` (primary key example)
        *   `AddColumn(_T("group_id"), &m_group_id, SQL_INTEGER);`
        *   `AddColumn(_T("opening_reg_date"), &m_opening_reg_date, SQL_TIMESTAMP);`
        *   `AddColumn(_T("note"), &m_note, SQL_VARCHAR, true, false, 255);` (nullable string example)
*   **`GetDefaultConnect()`:**
    *   Return your project's default database connection string (can be a placeholder initially).
*   **`GetDefaultSQL()`:**
    *   Return `_T("[M_GROUP_NOTES]");`.
*   **`DoFieldExchange(CFieldExchange* pFX)`:**
    *   `pFX->SetFieldType(CFieldExchange::outputColumn);`
    *   Use `RFX_Long()`, `RFX_Date()`, `RFX_Int()`, `RFX_Text()` for each field.
        *   Example: `RFX_Long(pFX, _T("[pk]"), m_pk);`
    *   For nullable fields, call `SetFieldNull()` after the `RFX_` calls.
        *   Example: `SetFieldNull(&m_note, IsNullNote(), pFX);`
*   **Getter/Setter Implementations:**
    *   Implement the functions declared in the `.h` file.
    *   Setters should also update the corresponding `m_bIsNull...` flag (managed by `AlterableRecordset`'s `AddColumn` and `SetNull` helpers).

## Step 3: Objects (`GroupNotesRecord.h/.cpp`, `GroupNotes.h/.cpp`, and changes to existing Group/Factory files)

### File: `GroupNotesRecord.h` (New)

*   **Class Definition:** `GroupNotesRecord` inheriting from `AlterableRecord`.
*   **Forward Declarations:** `GroupMain`, `ARSMGroupNotes`.
*   **Constructor:** `GroupNotesRecord(GroupMain* pGroupMain, const ATime& tmEffectDate, long lRecordIdentifier, DbState eDbState);`
*   **Key Getters:** `GetGroupId() const;`
*   **Underlying Record Methods:** `IsUnderlyingRecordSet() const;`, `SetUnderlyingRecord(ARSMGroupNotes* pRsGroupNotes);`, `CopyUnderlyingRecord(ARSMGroupNotes* pRsGroupNotesDest);`
*   **Field Getters/Setters:** For all `M_GROUP_NOTES` fields.
*   **Member Variables:** Store data for each field, `GroupMain* m_pGroupMain;`.

### File: `GroupNotesRecord.cpp` (New)

*   Implement Constructor.
*   Implement `GetGroupId()`: Delegate to `m_pGroupMain->GetGroupId()`.
*   Implement `SetUnderlyingRecord()`: Copy data from `ARSMGroupNotes` parameter to member variables. Set `DbState`, `Pk`, `EffectDate`.
*   Implement `CopyUnderlyingRecord()`: Copy data from member variables to `ARSMGroupNotes` parameter.
*   Implement Field Getters/Setters.

### File: `GroupNotes.h` (New)

*   **Class Definition:** `GroupNotes` inheriting from `AlterableObject`.
*   **Forward Declarations:** `Group`, `OpeningStatus`, `User`.
*   **Constructor:** `GroupNotes(Group* pGroup, DbState eDbState, bool bRawRecord = false);`
*   **Parent Access:** `GetGroup() const;`
*   **Key Info:** `GetGroupId() const;`, `GetEffectDate() const;`
*   **Field Accessors (Getters):** For all fields. For foreign key IDs (like `opening_status`, `user_id`), return pointers to objects (e.g., `OpeningStatus* GetOpeningStatus() const;`).
*   **Field Mutators (Setters):** For all fields. For foreign key objects, accept object pointers (e.g., `void SetOpeningStatus(OpeningStatus* pStatus);`).
*   **`AlterableObject` Overrides:** `SaveChanges()`, `Validate()`, `Delete()`, `IsChanged()`, `IsNew()`, `IsDeleted()`, `IsUnderlyingRecordSet()`, `SetUnderlyingRecord(ARSMGroupNotes*)`, `SetUnderlyingRecordFromObject(AlterableRecord*)`, `CopyUnderlyingRecord(ARSMGroupNotes*)`, `GetPk()`, `SetPk()`.
*   **Protected DB Methods:** `Insert()`, `Update()`, `LogicalDelete()`, `PhysicalDelete()`.
*   **Member Variables:** `Group* m_pGroup;`, `ARSMGroupNotes m_rsUnderlyingRecord;`, pointers for related objects like `OpeningStatus* m_pOpeningStatus;`.

### File: `GroupNotes.cpp` (New)

*   Implement Constructor: Initialize `m_rsUnderlyingRecord` with `ContextManager::GetProcess()->GetDBPtr()`.
*   Implement Getters: For simple fields, get from `m_rsUnderlyingRecord`. For object fields (e.g., `OpeningStatus`), use lazy loading: if `m_pOpeningStatus` is null and ID in `m_rsUnderlyingRecord` is set, fetch using `ObjectFactory`.
*   Implement Setters: Update `m_rsUnderlyingRecord` and set `m_bChanged = true;`. For object fields, store the pointer and update the ID in `m_rsUnderlyingRecord`.
*   Implement `SaveChanges()`: Call `Validate()`. If new, call `Insert()`. Else if changed, call `Update()`.
*   Implement `Validate()`: Add checks for required fields (e.g., `group_id`, `opening_effec_date`, `opening_status`).
*   Implement `Delete()`: Call `PhysicalDelete()` or `LogicalDelete()`.
*   Implement `Insert()`, `Update()`, `LogicalDelete()`, `PhysicalDelete()`: Use `m_rsUnderlyingRecord.ANew()`, `AEdit()`, `AUpdate()`, `ADelete()`. Call `CopyUnderlyingRecord(&m_rsUnderlyingRecord)` before `AUpdate()` in `Insert()` and `Update()`.
*   Implement `SetUnderlyingRecord()`: Copy data from passed `ARSMGroupNotes` to `m_rsUnderlyingRecord`. Reset cached object pointers (`m_pOpeningStatus`, etc.).
*   Implement `SetUnderlyingRecordFromObject()`: Cast `AlterableRecord*` to `GroupNotesRecord*`, then copy data to `m_rsUnderlyingRecord`. Reset cached object pointers.
*   Implement `CopyUnderlyingRecord()`: Copy data from `m_rsUnderlyingRecord` to the destination `ARSMGroupNotes`.

### File: `GroupMain.h` (Changes)

*   **Forward Declare:** `GroupNotesRecord`.
*   **Add Methods:** `TIterator<GroupNotesRecord*> GetGroupNotesRecords();`, `GroupNotesRecord* AddNewGroupNotesRecord(GroupNotesPrivateKey*, const ATime& tmEffecDate);`, `void AddGroupNotesRecord(GroupNotesRecord* pGroupNotesRecord);`.
*   **Add Member:** `TObjectList<GroupNotesRecord*> m_lstGroupNotesRecord;`.
*   **Add Member:** `bool m_bLoadGroupNotesRecords;`.
*   **Add Method:** `void LoadGroupNotesRecords();`.

### File: `GroupMain.cpp` (Changes)

*   **Includes:** `ARSMGroupNotes.h`, `GroupNotesRecord.h`.
*   **Constructor:** Initialize `m_bLoadGroupNotesRecords = false;`.
*   **`RefreshUpdatedTable()`:** Add case for `ARSMGroupNotes().GetTableName()` to set `m_bLoadGroupNotesRecords = false;`.
*   **Implement `GetGroupNotesRecords()`:** Call `LoadGroupNotesRecords();` then return iterator.
*   **Implement `AddNewGroupNotesRecord()`:** Use `ObjectFactory::GetInstance()->Groups()->GetGroupNotesRecord(...)` or `CreateGroupNotesRecord(...)`. Add to `m_lstGroupNotesRecord`.
*   **Implement `AddGroupNotesRecord()`:** Add to `m_lstGroupNotesRecord`.
*   **Implement `LoadGroupNotesRecords()`:** If `!m_bLoadGroupNotesRecords`, open `ARSMGroupNotes` for `GetGroupId()`. Loop, create `GroupNotesRecord` via `ObjectFactory`, call `pRecord->SetUnderlyingRecord(&rsMGroupNotes)`, add to `m_lstGroupNotesRecord`. Set flag to true.

### File: `Group.h` (Changes)

*   **Forward Declare:** `GroupNotes`, `GroupNotesRecord`.
*   **Add Methods:** `TIterator<GroupNotes*> GetGroupNotess(...)`, `GroupNotes* AddNewGroupNotes(...)`, `void AddGroupNotes(GroupNotes*)`, `void RemoveGroupNotes(GroupNotes*)`.
*   **Add Member:** `TObjectList<GroupNotes*> m_lstGroupNotes;`.
*   **Add Member:** `bool m_bLoadGroupNotesList;`.
*   **Add Method:** `void LoadGroupNotes();`.

### File: `Group.cpp` (Changes)

*   **Includes:** `GroupNotes.h`, `GroupNotesRecord.h`.
*   **Constructor:** Initialize `m_bLoadGroupNotesList = false;`.
*   **Implement `GetGroupNotess()`:** Call `LoadGroupNotes();` then return iterator (apply filters if needed).
*   **Implement `AddNewGroupNotes()`:** Use `ObjectFactory::GetInstance()->Groups()->CreateGroupNotes(this, DbState::NEW)`. Set default values (e.g., `OpeningEffecDate`, `OpeningRegDate`, `OpeningStatus`, `OpeningRef`, `ClosingEffecDate`). Add to `m_lstGroupNotes`.
*   **Implement `AddGroupNotes()`:** Add to `m_lstGroupNotes`.
*   **Implement `RemoveGroupNotes()`:** Remove from list, call `pGroupNotes->Delete()` if not new.
*   **Implement `LoadGroupNotes()`:** If `!m_bLoadGroupNotesList`, iterate `GetGroupMain()->GetGroupNotesRecords()`. For each `GroupNotesRecord`, get/create `GroupNotes` object via `ObjectFactory`, call `pNotes->SetUnderlyingRecordFromObject(pRecord)`, add to `m_lstGroupNotes`. Set flag to true.

### File: `IGroupFactory.h` (Changes)

*   **Forward Declare:** `GroupNotes`, `GroupNotesRecord`.
*   **Declare Methods:**
    *   `virtual GroupNotes* GetGroupNotes(Group* pGroup, DbState eDbState = DbState::UNDEFINED) = 0;`
    *   `virtual GroupNotesRecord* GetGroupNotesRecord(GroupMain* pGroupMain, const ATime& tmEffectDate, long lRecordIdentifier, DbState eDbState = DbState::UNDEFINED) = 0;`
    *   `virtual GroupNotes* CreateGroupNotes(Group* pGroup, DbState eDbState = DbState::UNDEFINED) = 0;`
    *   `virtual GroupNotesRecord* CreateGroupNotesRecord(GroupMain* pGroupMain, const ATime& tmEffectDate, long lRecordIdentifier, DbState eDbState = DbState::UNDEFINED) = 0;`

### File: `GroupFactory.cpp` (Changes)

*   **Includes:** `GroupNotes.h`, `GroupNotesRecord.h`.
*   **Implement `GetGroupNotes()`:** (Placeholder for caching logic) Call `CreateGroupNotes(...)`.
*   **Implement `GetGroupNotesRecord()`:** (Placeholder for caching logic) Call `CreateGroupNotesRecord(...)`.
*   **Implement `CreateGroupNotes()`:** `return new GroupNotes(pGroup, eDbState);` (add to `AlisMemoryManager`).
*   **Implement `CreateGroupNotesRecord()`:** `return new GroupNotesRecord(pGroupMain, tmEffectDate, lRecordIdentifier, eDbState);` (add to `AlisMemoryManager`).

## Step 4: API (`AUpdateGroupNotes.h` & `AUpdateGroupNotes.cpp`)

### File: `AUpdateGroupNotes.h` (New)

*   **`AUpdateGroupNotesArg` Class:** Inherit `AXMLProcedureArg`. Members: `m_lGroupId`, `m_sNote`, `m_tmOpeningEffecDate`, `m_sOpeningRef`.
*   **`AUpdateGroupNotesRet` Class:** Inherit `AXMLProcedureRet`. Member: `m_lGroupNotesPk`.
*   **`AUpdateGroupNotesLogic` Class:** Inherit `AXMLProcedure`.
    *   **Constructor:** `AUpdateGroupNotesLogic(AUpdateGroupNotesArg& arg, AUpdateGroupNotesRet& ret);`
    *   **Overrides:** `GetXMLInput()`, `Process()`, `Execute()`.
    *   **Members:** `AUpdateGroupNotesArg& m_arg;`, `AUpdateGroupNotesRet& m_ret;`, `Group* m_pGroup;`.
*   **Exported API Function:** `extern "C" DllExport int AUpdateGroupNotes(AUpdateGroupNotesArg& arg, AUpdateGroupNotesRet& ret);`
*   **Exported Factory Functions:** `GetAUpdateGroupNotesArg()`, `GetAUpdateGroupNotesRet()`.

### File: `AUpdateGroupNotes.cpp` (New)

*   **Includes:** `"stdafx.h"`, `"AUpdateGroupNotes.h"`, `ObjectFactory.h`, `Group.h`, `GroupNotes.h`, `OpeningStatus.h`, `ContextManager.h`, `ApiErrorMessages.h`.
*   **Implement API Entry Point `AUpdateGroupNotes(...)`:** Create `AUpdateGroupNotesLogic logic(arg, ret); return logic.Go();`.
*   **Implement Factory Functions:** `return new AUpdateGroupNotesArg();`, `return new AUpdateGroupNotesRet();`.
*   **Implement `AUpdateGroupNotesArg` Constructor.**
*   **Implement `AUpdateGroupNotesRet` Constructor.**
*   **Implement `AUpdateGroupNotesLogic` Constructor:** Call base `AXMLProcedure` constructor, initialize members.
*   **Implement `GetXMLInput()`:**
    *   Use `m_xmlInInterface.GetValue(_T("GroupId"), m_arg.m_lGroupId);`
    *   Use `m_xmlInInterface.GetValue(_T("Note"), m_arg.m_sNote);`
    *   Optionally get `OpeningEffecDate`, `OpeningRef`. If `OpeningEffecDate` not provided, use `m_arg.m_csTransDetails.m_tmTransEffDate`.
    *   Fetch `Group`: `m_pGroup = ObjectFactory::GetInstance()->Groups()->GetGroup(m_arg.m_lGroupId, m_arg.m_csTransDetails.m_tmTransEffDate);`.
    *   Error handling: `RaiseError(ApiErrorMessages::GetInstance()->MissingArgument.Format(...));`.
*   **Implement `Process()`:**
    *   Check if `m_pGroup` is valid.
    *   `GroupNotes* pGroupNotes = m_pGroup->AddNewGroupNotes();`.
    *   `pGroupNotes->SetNote(m_arg.m_sNote);`.
    *   If `m_arg.m_tmOpeningEffecDate` is set, `pGroupNotes->SetOpeningEffecDate(m_arg.m_tmOpeningEffecDate);`.
    *   If `m_arg.m_sOpeningRef` is set, `pGroupNotes->SetOpeningRef(m_arg.m_sOpeningRef);` else `pGroupNotes->SetOpeningRef(GetActivityRef(true));`.
    *   `pGroupNotes->SetUserId(ContextManager::GetProcess()->GetUserId());`.
    *   `pGroupNotes->SaveChanges();`.
    *   `m_ret.m_lGroupNotesPk = pGroupNotes->GetPk();`.
*   **Implement `Execute()`:** Can be empty.

**Remember to add new files to your project/solution and ensure include paths are correct.**
