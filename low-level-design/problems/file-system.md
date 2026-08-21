# File System

**Difficulty:** Medium | **Concepts:** Abstract base class, tree data structure, Map-based children, parent pointer, path resolution helpers, cycle detection

---

## Understanding the Problem

Design an in-memory file system supporting standard operations: create files and folders, delete, list, get, rename, and move. Paths are absolute Unix-style strings (`/home/user/notes.txt`). Files store string content. Folders contain other files and folders.

---

## Requirements Clarification

**Core questions:**
- Is the root fixed? → Yes, root is `/`; it always exists and cannot be deleted or moved.
- Absolute or relative paths? → Absolute only (always starts with `/`). No `..` or symlinks.
- What if you create a file at a path whose parent folder doesn't exist? → Throw an error.
- What happens on name collision (creating a file where one already exists)? → Throw AlreadyExistsException.
- Deletion behavior for non-empty folders? → Recursive delete (deletes all children).
- Case sensitive? → Yes.

**Final Requirements:**
```
Operations:
  createFile(path, content) → File
  createFolder(path) → Folder
  delete(path)
  list(path) → List<FileSystemEntry>
  get(path) → FileSystemEntry
  rename(path, newName)
  move(srcPath, destPath)

Constraints:
  - Root "/" always exists; cannot delete/move/rename root
  - Paths are absolute (start with "/")
  - Name collisions throw AlreadyExistsException
  - Missing paths throw NotFoundException
  - Moving a folder into itself or a descendant is a cycle — throw InvalidPathException

Out of scope: permissions, symlinks, disk persistence, relative paths, concurrent access
  (thread-safety can be asked as an extensibility question)
```

---

## Core Entities

Candidate nouns: file, folder, directory, path, entry, content, name.

**Path** — a string. Not an entity.

**Content** — a string stored in File. Not an entity.

**Name** — a string field. Not an entity.

**File** — stores content, has a name, lives inside a folder. **Entity.**

**Folder** — contains children (files and folders), has a name, lives inside a folder (except root). **Entity.**

The key observation: File and Folder share several properties — name, a reference to their parent folder, and the ability to report their path. Rather than duplicating these in both classes, extract a common abstract base class.

**FileSystemEntry** — abstract base class. Holds name and parent pointer; computes path. **Entity.**

**FileSystem** — the entry point. Holds root, exposes all public operations. **Entity.**

| Entity | Responsibility |
|--------|---------------|
| **FileSystem** | Entry point. Path parsing, operation dispatch. |
| **FileSystemEntry** | Abstract base: name, parent pointer, path computation. |
| **File** | Extends FileSystemEntry. Stores content. `isDirectory() = false`. |
| **Folder** | Extends FileSystemEntry. Manages children Map. `isDirectory() = true`. |

---

## Class Design

### FileSystemEntry (Abstract)

```
abstract class FileSystemEntry:
    - name: string
    - parent: Folder?          // null for root

    + FileSystemEntry(name)
    + getName() → string
    + setName(name)
    + getParent() → Folder?
    + setParent(Folder?)
    + getPath() → string       // computed by walking parent pointers
    + isDirectory() → boolean  // abstract — File returns false, Folder returns true
```

**Why a parent pointer instead of storing the path as a string?**

If you stored the path string on each entry, renaming or moving a folder with 10,000 descendants would require updating every single descendant's path string. With a parent pointer, `getPath()` computes the path dynamically by walking up the tree — O(depth), which is logarithmic for balanced trees. Rename only updates the name on one node; all descendants automatically get the right path on next call.

**`getPath()` implementation:**
```
getPath():
    if parent == null:
        return name    // root folder returns "/" (initialized with name="/")

    parentPath = parent.getPath()
    if parentPath == "/":
        return "/" + name          // avoid "//home"
    else:
        return parentPath + "/" + name
```

A file at `/home/user/notes.txt` asks `user` for its path → `/home/user`. Returns `/home/user` + `/` + `notes.txt` = `/home/user/notes.txt`. The root special case prevents double slashes.

### File

```
class File extends FileSystemEntry:
    - content: string

    + File(name, content)
    + getContent() → string
    + setContent(content)
    + isDirectory() → false
```

File is a data holder with no interesting logic beyond its content.

### Folder

```
class Folder extends FileSystemEntry:
    - children: Map<string, FileSystemEntry>   // key = entry name

    + Folder(name)
    + isDirectory() → true
    + addChild(entry) → boolean
    + removeChild(name) → FileSystemEntry?
    + getChild(name) → FileSystemEntry?
    + hasChild(name) → boolean
    + getChildren() → List<FileSystemEntry>
```

**Why `Map<string, FileSystemEntry>` not `List<FileSystemEntry>`?**

Looking up children by name (hasChild, getChild) is the most common operation. A List requires O(n) linear scan. A Map gives O(1) lookup. At tens-of-thousands of entries, this difference is significant. The key is the entry's name — names within a folder must be unique (that's what the file system guarantees).

**Folder methods maintain bidirectional consistency:**
```
addChild(entry):
    if entry == null: return false
    if children.containsKey(entry.getName()): return false  // collision
    children.put(entry.getName(), entry)
    entry.setParent(this)    // maintain the back-reference
    return true

removeChild(name):
    entry = children.remove(name)
    if entry != null:
        entry.setParent(null)    // clear the back-reference
    return entry
```

**Why does `addChild` call `entry.setParent(this)`?** The parent pointer must stay in sync with the children map. If you added a child without setting its parent, `getPath()` would return the wrong path. If you removed a child without clearing its parent, the orphaned entry would still report the old parent path.

**`getChildren()` returns a copy:**
```
getChildren():
    return new List(children.values())
```
Returns a snapshot, not the live map. Callers can iterate without risking ConcurrentModificationException, and they can't accidentally modify the internal map.

### FileSystem

```
class FileSystem:
    - root: Folder

    + FileSystem()    // creates root = Folder("/")
    + createFile(path, content) → File
    + createFolder(path) → Folder
    + delete(path)
    + list(path) → List<FileSystemEntry>
    + get(path) → FileSystemEntry
    + rename(path, newName)
    + move(srcPath, destPath)
```

FileSystem's job is path parsing and delegation. All path manipulation goes through three private helpers: `resolvePath`, `resolveParent`, `extractName`.

---

## Implementation

### Path Resolution Helpers

All public methods delegate the messy path-string-to-tree-node conversion to these three helpers, keeping the public methods focused on their actual job.

**`resolvePath(path)` — walk the tree to find the entry:**
```
resolvePath(path):
    if path == null || path is empty: throw InvalidPathException
    if !path.startsWith("/"): throw InvalidPathException("Path must be absolute")
    if path == "/": return root

    parts = path.substring(1).split("/")    // "/home/user/docs" → ["home", "user", "docs"]
    current = root
    for part in parts:
        if part is empty: throw InvalidPathException("Invalid path: consecutive slashes")
        if !current.isDirectory(): throw NotADirectoryException
        child = current.getChild(part)
        if child == null: throw NotFoundException("Path not found: " + path)
        current = child
    return current
```

Intentionally simple: splits on `/` and walks forward. No relative paths (`../`), no symlinks — requirements explicitly excluded them.

**`resolveParent(path)` — find the containing folder:**
```
resolveParent(path):
    if path == "/": throw InvalidPathException("Root has no parent")
    lastSlash = path.lastIndexOf("/")
    parentPath = (lastSlash == 0) ? "/" : path.substring(0, lastSlash)
    parent = resolvePath(parentPath)
    if !parent.isDirectory(): throw NotADirectoryException
    return parent
```

For `/home/user/notes.txt`, `lastSlash=10`, `parentPath="/home/user"`, returns the Folder at `/home/user`. For `/readme.txt`, `lastSlash=0`, `parentPath="/"`, returns root.

**`extractName(path)` — last component of a path:**
```
extractName(path):
    lastSlash = path.lastIndexOf("/")
    return path.substring(lastSlash + 1)
```

### `createFile`

```
createFile(path, content):
    if path == "/": throw InvalidPathException("Cannot create file at root")
    parent = resolveParent(path)
    fileName = extractName(path)
    if parent.hasChild(fileName): throw AlreadyExistsException(...)
    file = File(fileName, content)
    parent.addChild(file)    // also sets file.parent = parent
    return file
```

### `createFolder`

```
createFolder(path):
    if path == "/": throw InvalidPathException("Cannot create folder at root")
    parent = resolveParent(path)
    folderName = extractName(path)
    if parent.hasChild(folderName): throw AlreadyExistsException(...)
    folder = Folder(folderName)
    parent.addChild(folder)
    return folder
```

### `delete`

```
delete(path):
    if path == "/": throw InvalidPathException("Cannot delete root")
    parent = resolveParent(path)
    name = extractName(path)
    entry = parent.removeChild(name)    // removes from map, clears parent pointer
    if entry == null: throw NotFoundException(...)
    // Recursive deletion: if entry is a Folder, its children are now unreachable
    // and will be garbage collected. No explicit recursion needed in GC languages.
```

### `rename`

Rename is the trickiest operation because children are keyed by name in the Map. You can't just call `setName()` — the old key still exists in the map, now pointing to an entry with a different name.

```
rename(path, newName):
    if path == "/": throw InvalidPathException("Cannot rename root")
    if newName is null || newName is empty || newName.contains("/"): throw InvalidPathException("Invalid name")

    parent = resolveParent(path)
    oldName = extractName(path)
    if !parent.hasChild(oldName): throw NotFoundException(...)
    if parent.hasChild(newName): throw AlreadyExistsException(...)

    entry = parent.removeChild(oldName)    // remove under old key; clears parent pointer
    entry.setName(newName)                 // update the name
    parent.addChild(entry)                 // re-insert under new key; sets parent pointer
```

The three-step dance: remove → rename → re-insert. All children of a renamed folder automatically report the correct new path because `getPath()` is computed dynamically via parent pointers — not cached as a string.

### `move`

```
move(srcPath, destPath):
    if srcPath == "/": throw InvalidPathException("Cannot move root")

    srcParent = resolveParent(srcPath)
    srcName = extractName(srcPath)
    entry = srcParent.getChild(srcName)
    if entry == null: throw NotFoundException("Source not found: " + srcPath)

    destParent = resolveParent(destPath)
    destName = extractName(destPath)

    // Cycle detection: can't move a folder into itself or a descendant
    if entry.isDirectory():
        current = destParent
        while current != null:
            if current == entry: throw InvalidPathException("Cannot move folder into itself")
            current = current.getParent()

    // Collision check at destination
    if destParent.hasChild(destName): throw AlreadyExistsException(...)

    // Perform the move
    srcParent.removeChild(srcName)    // detaches from source
    entry.setName(destName)           // rename if destName differs
    destParent.addChild(entry)        // attaches at destination; sets entry.parent = destParent
```

**Why is cycle detection necessary?** If you move `/home` into `/home/user/stuff`, you'd create an impossible loop where `/home` is both an ancestor and a descendant of `/home/user`. The cycle detection walks up from `destParent` toward the root; if it ever hits the entry being moved, the move would create a cycle and is rejected.

```
// Cycle detection trace for move("/home", "/home/user/stuff"):
current = destParent = user/stuff folder
  current.getParent() = user folder
    current.getParent() = home folder = entry → CYCLE DETECTED! Reject.
```

---

## Verification Trace

**Build folder structure:**
```
Initial: root = Folder("/"), root.children = {}

createFolder("/home"):
  resolveParent → root
  extractName → "home"
  root.hasChild("home") → false
  Create Folder("home"), root.addChild(home) → home.parent = root
  State: root.children = {"home" → Folder}

createFolder("/home/user"):
  resolveParent("/home/user") → resolvePath("/home") → home folder
  extractName → "user"
  Create Folder("user"), home.addChild(user) → user.parent = home
  State: home.children = {"user" → Folder}, user.parent = home
```

**Create and move a file:**
```
createFile("/home/user/notes.txt", "hello world"):
  parent = home/user folder
  Create File("notes.txt", "hello world"), user.addChild(notes)
  State: user.children = {"notes.txt" → File}, notes.parent = user

notes.getPath():
  parent = user → user.getPath():
    parent = home → home.getPath():
      parent = root → root.getPath():
        parent = null → return "/"
      parentPath = "/" → return "/" + "home" = "/home"
    parentPath = "/home" → return "/home" + "/" + "user" = "/home/user"
  parentPath = "/home/user" → return "/home/user" + "/" + "notes.txt" = "/home/user/notes.txt" ✓

move("/home/user/notes.txt", "/home/notes.txt"):
  srcParent = user, srcName = "notes.txt"
  entry = File("notes.txt")
  destParent = home, destName = "notes.txt"
  Cycle check: File is not a directory → skip
  home.hasChild("notes.txt") → false
  user.removeChild("notes.txt") → removes, notes.parent = null
  entry.setName("notes.txt") → name unchanged
  home.addChild(notes) → notes.parent = home

  notes.getPath() → "/home" + "/" + "notes.txt" = "/home/notes.txt" ✓
```

**Error handling:**
```
createFile("/home/notes.txt", "duplicate"):
  home.hasChild("notes.txt") → true → Throws AlreadyExistsException ✓

delete("/nonexistent"):
  resolveParent → root
  root.removeChild("nonexistent") → null
  Throws NotFoundException ✓

move("/home", "/home/user/stuff"):
  entry = home folder
  destParent = user (inside home)
  Cycle check: walk from user up → user.parent = home = entry → CYCLE DETECTED
  Throws InvalidPathException("Cannot move folder into itself") ✓
```

---

## Extensibility

### "How would you make this file system thread-safe?"

**The race condition:** Two threads call `createFile("/home/notes.txt", ...)` simultaneously. Both call `hasChild("notes.txt")` and see false. Both proceed. One calls `addChild` — the file is added. The other calls `addChild` — depending on HashMap implementation, this overwrites the first file silently, or corrupts the map. Silent data loss.

This is a check-then-act race: the check (`hasChild`) and the action (`addChild`) aren't atomic.

**Simple fix — coarse-grained lock on FileSystem:**
```
createFile(path, content):
    synchronized(this):
        // entire method body runs while holding the lock
        parent = resolveParent(path)
        fileName = extractName(path)
        if parent.hasChild(fileName): throw AlreadyExistsException
        file = File(fileName, content)
        parent.addChild(file)
        return file
```

Now only one thread can execute any FileSystem method at a time. Simple and correct. Tradeoff: two threads creating files in completely different folders (`/alice/file1.txt` and `/bob/file2.txt`) still block each other even though they're not touching the same data.

**Better fix — fine-grained lock per folder:**
```
createFile(path, content):
    parent = resolveParent(path)
    fileName = extractName(path)
    synchronized(parent):    // lock just the parent folder
        if parent.hasChild(fileName): throw AlreadyExistsException
        file = File(fileName, content)
        parent.addChild(file)
    return file
```

Now threads operating on different folders proceed in parallel.

**Deadlock risk with `move`:** Move touches two folders (source parent and destination parent). If Thread A moves from `/alice` and Thread B moves from `/bob`, and both need both locks:
- Thread A: locks `/alice`, waits for `/bob`
- Thread B: locks `/bob`, waits for `/alice`
- Deadlock!

**Fix — lock ordering by path string (alphabetical):**
```
move(srcPath, destPath):
    srcParent = resolveParent(srcPath)
    destParent = resolveParent(destPath)

    // Always lock in alphabetical order by path
    firstLock = srcParent.getPath() < destParent.getPath() ? srcParent : destParent
    secondLock = srcParent.getPath() < destParent.getPath() ? destParent : srcParent

    synchronized(firstLock):
        synchronized(secondLock):
            // both folders locked — safe to modify both
            srcName = extractName(srcPath)
            entry = srcParent.getChild(srcName)
            // ... cycle check, collision check ...
            srcParent.removeChild(srcName)
            entry.setName(extractName(destPath))
            destParent.addChild(entry)
```

With consistent ordering, Thread A and Thread B both try to lock `/alice` first (comes before `/bob` alphabetically). One gets it, the other waits. No circular wait, no deadlock.

**Read-write lock for read-heavy workloads:**
Operations like `get` and `list` only read data; they don't modify anything. A standard lock makes readers block each other unnecessarily. A read-write lock allows concurrent readers:
```
get(path):
    readLock.acquire()
    try: return resolvePath(path)
    finally: readLock.release()

createFile(path, content):
    writeLock.acquire()
    try: // create logic
    finally: writeLock.release()
```

Multiple threads can read simultaneously. Any write gets exclusive access.

For interview scope: mention coarse-grained locking first, note fine-grained exists for throughput, mention the deadlock risk with `move`, and explain lock ordering as the fix.

### "How would you add search functionality?"

**Recursive traversal — O(n):**
```
search(startFolder, searchName):
    results = []
    searchHelper(startFolder, searchName, results)
    return results

searchHelper(folder, searchName, results):
    for entry in folder.getChildren():
        if entry.getName() == searchName:
            results.add(entry)
        if entry.isDirectory():
            searchHelper(entry, searchName, results)
```

Simple, but O(n) where n is the total number of entries. For large trees, every search scans everything.

**Index for O(1) search by name:**
```
class FileSystem:
    - root: Folder
    - nameIndex: Map<string, List<FileSystemEntry>>
```

When creating an entry, add it to the index. When deleting, remove it. When renaming, move from old name's list to new name's list:
```
// In rename():
nameIndex.get(oldName).remove(entry)
entry.setName(newName)
parent.addChild(entry)
nameIndex.computeIfAbsent(newName, _ -> []).add(entry)

// Search becomes O(1):
search(name):
    return nameIndex.containsKey(name) ? nameIndex.get(name) : []
```

**Further search extensions:**
- **Prefix search** — replace Map with a Trie. "config*" traverses to "config" node and collects all descendants.
- **Wildcard patterns** — maintain secondary index by extension for `*.txt`; fall back to traversal for complex globs.
- **Content search** — inverted index mapping words to files containing them. Essentially a mini search engine — overkill unless specifically asked.

For interview scope: basic recursive traversal + mention the name index optimization demonstrates understanding of performance tradeoffs.

---

## What Interviewers Look For

| Level | Bar |
|-------|-----|
| Junior | FileSystem entry point + File + Folder as tree nodes. Basic operations with path parsing. Map for children (or recognized as ideal when asked). Reject non-existent paths, handle name collisions. May not think of FileSystemEntry base class independently — fine. |
| Mid-level | Proactively extract FileSystemEntry base class (recognize duplication in name + parent + getPath). Parent pointer vs. stored path tradeoff explained. `move()` handles parent pointer updates correctly. Might need a hint about cycle detection, but implements it once prompted. |
| Senior | Parent pointer vs. stored path discussed proactively: "if I stored paths as strings, renaming a folder with thousands of descendants requires updating all their paths." Cycle detection in `move()` comes up naturally. Thread-safety concerns discussed: check-then-act race in `createFile`, need to lock two folders atomically for `move`, lock ordering prevents deadlock. Search index optimization mentioned. |

---

## Practice Questions

1. Why does renaming a folder in this design not require updating any child paths? What data structure change would make rename O(n) in the number of descendants, and why is the parent pointer approach O(1)?
2. Trace the cycle detection for `move("/home/user", "/home/user/backup")`. Walk step by step through the while loop. At what iteration is the cycle detected?
3. `addChild` calls `entry.setParent(this)`, and `removeChild` calls `entry.setParent(null)`. What specific bug would occur in `move()` if `removeChild` forgot to clear the parent pointer?
4. The `getChildren()` method returns a copy of `children.values()` rather than the live collection. Why? What specific failure would occur if it returned the live Map values and a caller modified the returned collection?
5. Two threads simultaneously call `createFolder("/home/alice")` and `createFolder("/home/bob")`. With fine-grained locking (per folder), do these operations block each other? Explain why or why not. Now both call `move("/home/alice/file.txt", "/home/bob/file.txt")` simultaneously — do those block each other?

## One-Line Summary

File System's key insight is the abstract `FileSystemEntry` base class with a parent pointer — it eliminates duplication between File and Folder, enables O(depth) dynamic path computation (so rename/move never requires updating descendants), and the Map-based children collection provides O(1) lookup while the rename operation must explicitly remove-rename-reinsert to keep the map key in sync with the entry name.
