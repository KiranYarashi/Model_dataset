# Core Suite Business Logic Developer Training - Hands-on Guide

This document provides step-by-step instructions for adding a new "Group Notes" feature.

## Step 1: Create of a new Database table `M_GROUP_NOTES`

**Reason:** This is the foundational step. The application needs a dedicated table in the database to persistently store all the information related to each group note. Without this table, there's nowhere to save or retrieve the notes data.

**Action:** This step is performed directly in your database management tool (e.g., SQL Server Management Studio).

**Hints:**
*   Ensure data types match the specification (e.g., `INT`, `DATETIME`, `VARCHAR`, `SMALLINT`).
*   Define `pk` as `IDENTITY` and `NOT NULL`.
*   Set `NOT NULL` constraints where specified.
*   Set `DEFAULT ((0))` for `discriminator`.

## Step 2: Record_Set (`ARSMGroupNotes.h` & `ARSMGroupNotes.cpp`)

**Reason:** The `AlterableRecordset` (and its derivative `ARSMGroupNotes`) provides a C++ class that directly maps to the `M_GROUP_NOTES` database table. It's used by the MFC framework to perform low-level database operations like fetching rows, inserting new rows, and updating existing ones. It's the primary C++ interface for raw data interaction with the table.

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

**Reason:** This step involves creating higher-level C++ objects that represent the "Group Notes" data and its associated business logic. This abstracts away the direct database interaction and provides a more structured, object-oriented way to work with the data.

### File: `GroupNotesRecord.h` (New) & `GroupNotesRecord.cpp` (New)

*   **Reason:** `GroupNotesRecord` acts as a lightweight data carrier or "record" object. It's typically used when loading lists of notes (e.g., all notes for a specific group) from the database. It holds the raw data from a row in `M_GROUP_NOTES` but doesn't contain complex business logic, making it efficient for collection loading. It often serves as an intermediary between the `ARSMGroupNotes` recordset and the full `GroupNotes` business object.
*   **Class Definition:** `GroupNotesRecord` inheriting from `AlterableRecord`.
*   **Forward Declarations:** `GroupMain`, `ARSMGroupNotes`.
*   **Constructor:** `GroupNotesRecord(GroupMain* pGroupMain, const ATime& tmEffectDate, long lRecordIdentifier, DbState eDbState);`
*   **Key Getters:** `GetGroupId() const;`
*   **Underlying Record Methods:** `IsUnderlyingRecordSet() const;`, `SetUnderlyingRecord(ARSMGroupNotes* pRsGroupNotes);`, `CopyUnderlyingRecord(ARSMGroupNotes* pRsGroupNotesDest);`
*   **Field Getters/Setters:** For all `M_GROUP_NOTES` fields.
*   **Member Variables:** Store data for each field, `GroupMain* m_pGroupMain;`.
*   **(In .cpp) Implement Constructor, GetGroupId(), SetUnderlyingRecord(), CopyUnderlyingRecord(), Field Getters/Setters.**

### File: `GroupNotes.h` (New) & `GroupNotes.cpp` (New)

*   **Reason:** `GroupNotes` is the core business object for a single group note. It encapsulates not only the data fields of a note but also the business rules, validation logic, and operations like saving, deleting, and managing relationships with other objects (e.g., `User`, `OpeningStatus`). This is the object your API and other business logic will primarily interact with.
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
*   **(In .cpp) Implement Constructor, Getters (with lazy loading for related objects), Setters, SaveChanges(), Validate(), Delete(), DB Methods, SetUnderlyingRecord(), SetUnderlyingRecordFromObject(), CopyUnderlyingRecord().**

### File: `GroupMain.h` (Changes) & `GroupMain.cpp` (Changes)

*   **Reason:** The `GroupMain` object often handles the loading and caching of a `Group`'s persistent data, including its collections of related child records. It needs to be modified to manage the collection of `GroupNotesRecord` objects associated with a `Group`. This allows efficient loading of all notes for a group.
*   **Forward Declare:** `GroupNotesRecord`.
*   **Add Methods:** `TIterator<GroupNotesRecord*> GetGroupNotesRecords();`, `GroupNotesRecord* AddNewGroupNotesRecord(GroupNotesPrivateKey*, const ATime& tmEffecDate);`, `void AddGroupNotesRecord(GroupNotesRecord* pGroupNotesRecord);`.
*   **Add Member:** `TObjectList<GroupNotesRecord*> m_lstGroupNotesRecord;`.
*   **Add Member:** `bool m_bLoadGroupNotesRecords;`.
*   **Add Method:** `void LoadGroupNotesRecords();`.
*   **(In .cpp) Initialize members, update `RefreshUpdatedTable()`, implement new methods for managing and loading `GroupNotesRecord`s.**

### File: `Group.h` (Changes) & `Group.cpp` (Changes)

*   **Reason:** The `Group` business object is the parent of `GroupNotes`. It needs to be modified to manage its collection of `GroupNotes` *business objects*. This includes methods to add new notes, retrieve existing notes (which will involve `GroupMain` and `GroupNotesRecord` for loading), and remove notes.
*   **Forward Declare:** `GroupNotes`, `GroupNotesRecord`.
*   **Add Methods:** `TIterator<GroupNotes*> GetGroupNotess(...)`, `GroupNotes* AddNewGroupNotes(...)`, `void AddGroupNotes(GroupNotes*)`, `void RemoveGroupNotes(GroupNotes*)`.
*   **Add Member:** `TObjectList<GroupNotes*> m_lstGroupNotes;`.
*   **Add Member:** `bool m_bLoadGroupNotesList;`.
*   **Add Method:** `void LoadGroupNotes();`.
*   **(In .cpp) Initialize members, implement new methods for managing and loading `GroupNotes` objects, often by converting `GroupNotesRecord`s into `GroupNotes` objects.**

### File: `IGroupFactory.h` (Changes) & `GroupFactory.cpp` (Changes)

*   **Reason:** The Factory pattern is used for creating objects. The `GroupFactory` (and its interface `IGroupFactory`) needs to be updated to know how to create instances of `GroupNotes` and `GroupNotesRecord`. This centralizes object creation, promotes consistency, and can be used for dependency injection or managing object lifecycles.
*   **Forward Declare:** `GroupNotes`, `GroupNotesRecord`.
*   **Declare Methods:**
    *   `virtual GroupNotes* GetGroupNotes(Group* pGroup, DbState eDbState = DbState::UNDEFINED) = 0;`
    *   `virtual GroupNotesRecord* GetGroupNotesRecord(GroupMain* pGroupMain, const ATime& tmEffectDate, long lRecordIdentifier, DbState eDbState = DbState::UNDEFINED) = 0;`
    *   `virtual GroupNotes* CreateGroupNotes(Group* pGroup, DbState eDbState = DbState::UNDEFINED) = 0;`
    *   `virtual GroupNotesRecord* CreateGroupNotesRecord(GroupMain* pGroupMain, const ATime& tmEffectDate, long lRecordIdentifier, DbState eDbState = DbState::UNDEFINED) = 0;`
*   **(In .cpp) Implement the new factory methods to instantiate `GroupNotes` and `GroupNotesRecord` objects.**

## Step 4: API (`AUpdateGroupNotes.h` & `AUpdateGroupNotes.cpp`)

**Reason:** This step creates the external Application Programming Interface (API) endpoint. This allows other parts of the system (or external applications) to interact with the "Group Notes" functionality, specifically to add a new note to a group. It defines the contract for how to call this feature, what inputs are expected (e.g., group ID, note text), and what output is returned (e.g., the PK of the newly created note).

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