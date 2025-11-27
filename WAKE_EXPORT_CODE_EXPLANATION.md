# Wake Export Code - Line by Line Explanation

## Overview
This document explains the code modifications that export wake point coordinates to a text file, organized by angle of attack (alpha) values.

---

## Part 1: Static Variables Declaration
**File:** `VSP_Solver.C` (Lines 11-15)

```cpp
// ===== My modification =======
// ---- New for text wake dump ----
static FILE * WakeTextFile = nullptr;
static int WakeTextFileFirstWrite = 1;  // Flag to track first write (truncate file)
// ---------------------------------
```

### Line-by-Line Explanation:

**Line 13:** `static FILE * WakeTextFile = nullptr;`
- **Purpose:** Declares a static file pointer to hold the handle for the wake text file
- **`static`:** Means this variable persists across function calls but is only accessible in this file
- **`FILE *`:** C file pointer type used for file I/O operations
- **`= nullptr`:** Initializes to null (no file open yet)
- **Why static:** We need this variable to persist between calls to `WriteOutAerothermalDatabaseSolution()`

**Line 14:** `static int WakeTextFileFirstWrite = 1;`
- **Purpose:** Flag to track if this is the first write in a sweep
- **`= 1`:** Starts as `true` (first write)
- **Usage:** 
  - When `1`: Open file in "w" mode (truncate/overwrite)
  - When `0`: Open file in "a" mode (append to existing file)
- **Why needed:** For alpha sweeps, we want to append each case's data, not overwrite

---

## Part 2: Opening the Text File and Writing Headers
**File:** `VSP_Solver.C` (Lines 32580-32600)

```cpp
// ===== My modification =======
// ---- open our text wake‐dump ----
{
    char wake_name[2048];
    sprintf( wake_name, "%s_wake.txt", FileName_ );
    // Open in write mode for first case (truncate), append mode for subsequent cases
    const char *mode = ( WakeTextFileFirstWrite ) ? "w" : "a";
    WakeTextFile = fopen( wake_name, mode );
    if ( !WakeTextFile )
        printf("** ERROR: Could not open %s for wake dump\n", wake_name );
    else {
        // Write header with alpha, mach, beta for this case
        fprintf( WakeTextFile, "# Alpha: % .6e  Mach: % .6e  Beta: % .6e\n", 
                 double(AngleOfAttack_/TORAD), Mach_, double(AngleOfBeta_/TORAD) );
        // Write column header for wake point coordinates
        fprintf( WakeTextFile, "x,y,z\n" );
        fflush( WakeTextFile );
    }
    WakeTextFileFirstWrite = 0;  // Next write will append
}
// ----------------------------------
```

### Line-by-Line Explanation:

**Line 32582:** `{`
- **Purpose:** Starts a code block to limit the scope of `wake_name` variable
- **Why:** `wake_name` is only needed here, so we limit its scope

**Line 32583:** `char wake_name[2048];`
- **Purpose:** Declares a character array to hold the filename
- **`[2048]`:** Buffer size (2048 characters) - enough for long file paths
- **Type:** `char` array (C-style string)

**Line 32584:** `sprintf( wake_name, "%s_wake.txt", FileName_ );`
- **Purpose:** Constructs the filename by inserting `FileName_` into the format string
- **`sprintf`:** String print function - writes formatted string to a buffer
- **`"%s_wake.txt"`:** Format string where `%s` will be replaced with `FileName_`
- **Example:** If `FileName_` is `"aircraft"`, result is `"aircraft_wake.txt"`
- **`FileName_`:** Member variable containing the base filename for this analysis

**Line 32586:** `const char *mode = ( WakeTextFileFirstWrite ) ? "w" : "a";`
- **Purpose:** Determines file open mode based on whether this is the first write
- **Ternary operator:** `condition ? value_if_true : value_if_false`
- **If `WakeTextFileFirstWrite == 1`:** `mode = "w"` (write mode - truncates file)
- **If `WakeTextFileFirstWrite == 0`:** `mode = "a"` (append mode - adds to file)
- **`const char *`:** Pointer to constant string (can't modify the string)

**Line 32587:** `WakeTextFile = fopen( wake_name, mode );`
- **Purpose:** Opens the file and stores the file pointer
- **`fopen`:** Standard C function to open a file
- **Parameters:**
  - `wake_name`: The filename to open
  - `mode`: "w" for write/truncate, "a" for append
- **Returns:** File pointer on success, `NULL` on failure
- **Stores result in:** `WakeTextFile` static variable

**Line 32588:** `if ( !WakeTextFile )`
- **Purpose:** Checks if file opening failed
- **`!WakeTextFile`:** True if `WakeTextFile` is `NULL` (file open failed)
- **Why check:** File might fail to open (permissions, disk full, etc.)

**Line 32589:** `printf("** ERROR: Could not open %s for wake dump\n", wake_name );`
- **Purpose:** Prints error message if file couldn't be opened
- **`printf`:** Standard output function
- **`%s`:** Format specifier for string (replaced with `wake_name`)
- **Note:** Execution continues even if file can't be opened (graceful degradation)

**Line 32590:** `else {`
- **Purpose:** Code block executed if file opened successfully

**Line 32592:** `fprintf( WakeTextFile, "# Alpha: % .6e  Mach: % .6e  Beta: % .6e\n", ...);`
- **Purpose:** Writes a header line with case parameters (alpha, Mach, Beta)
- **`fprintf`:** Formatted print to file (like `printf` but writes to file)
- **`WakeTextFile`:** File to write to
- **`"# Alpha: % .6e  Mach: % .6e  Beta: % .6e\n"`:** Format string
  - `#`: Comment marker (makes it easy to skip when parsing)
  - `% .6e`: Scientific notation with 6 decimal places, space for sign
  - `\n`: Newline character
- **Arguments:**
  - `double(AngleOfAttack_/TORAD)`: Converts angle from radians to degrees
    - `AngleOfAttack_`: Angle of attack in radians (member variable)
    - `TORAD`: Conversion constant (likely `PI/180.0`)
    - `double(...)`: Explicit cast to double
  - `Mach_`: Mach number (member variable)
  - `double(AngleOfBeta_/TORAD)`: Beta angle in degrees (side-slip angle)
- **Example output:** `# Alpha:  3.000000e+00  Mach:  1.000000e-01  Beta:  0.000000e+00`

**Line 32595:** `fprintf( WakeTextFile, "x,y,z\n" );`
- **Purpose:** Writes column header for the wake point coordinates
- **Format:** CSV-style header: `x,y,z`
- **Why:** Makes it clear what each column represents
- **Note:** This appears after the alpha/mach/beta header for each case

**Line 32596:** `fflush( WakeTextFile );`
- **Purpose:** Forces immediate write of buffered data to disk
- **`fflush`:** Flushes the file buffer
- **Why:** Ensures header is written before wake points are written (important for data integrity)
- **Without this:** Data might stay in memory buffer and not be written if program crashes

**Line 32598:** `WakeTextFileFirstWrite = 0;`
- **Purpose:** Sets flag to indicate first write is done
- **Effect:** Next time this function is called, file will open in append mode ("a")
- **Why:** Allows multiple cases (alpha sweep) to accumulate in same file

---

## Part 3: Writing Wake Point Coordinates
**File:** `Vortex_Trail.C` (Lines 970-1014)

```cpp
void VORTEX_TRAIL::WriteToFile(FILE *adb_file, FILE *WakeTextFile)
{
 
   int i, n, i_size, c_size, d_size;
   double x, y, z, s;
   
   // Sizeof int and double

   i_size = sizeof(int);
   c_size = sizeof(char);
   d_size = sizeof(double);

   n = NumberOfNodes();

   if ( TimeAccurate_ ) n = MIN( Time_ + 1, NumberOfNodes());

   if ( TE_Node_Region_Is_Concave_ ) n = 1;
   
   fwrite(&(TE_Node_), i_size, 1, adb_file);  // Kutta node
      
   s = double (SoverB_);
   
   fwrite(&(s), d_size, 1, adb_file); // S over B (span) 
   
   fwrite(&(n), i_size, 1, adb_file); // Number of nodes

   for ( i = 1 ; i <= n ; i++ ) {

      x = double (NodeList_[i].x());
      y = double (NodeList_[i].y());
      z = double (NodeList_[i].z());

      fwrite(&(x), d_size, 1, adb_file);
      fwrite(&(y), d_size, 1, adb_file);
      fwrite(&(z), d_size, 1, adb_file);

      // ====== My modification =========
      if ( WakeTextFile != nullptr ) {
          fprintf( WakeTextFile, "% .6e % .6e % .6e\n", x, y, z );
      }
      // ================================      

   }

}
```

### Line-by-Line Explanation:

**Line 970:** `void VORTEX_TRAIL::WriteToFile(FILE *adb_file, FILE *WakeTextFile)`
- **Purpose:** Function definition - writes wake data to both binary ADB file and text file
- **`VORTEX_TRAIL::`:** Member function of the `VORTEX_TRAIL` class
- **Parameters:**
  - `FILE *adb_file`: Binary ADB file (existing functionality)
  - `FILE *WakeTextFile`: Text file for wake points (new parameter)
- **Return:** `void` (nothing returned)

**Line 973:** `int i, n, i_size, c_size, d_size;`
- **Purpose:** Declares integer variables
- **`i`:** Loop counter
- **`n`:** Number of nodes to write
- **`i_size, c_size, d_size`:** Sizes of data types (for binary file writing)

**Line 974:** `double x, y, z, s;`
- **Purpose:** Declares floating-point variables
- **`x, y, z`:** Wake point coordinates
- **`s`:** Span location (S over B)

**Line 978:** `i_size = sizeof(int);`
- **Purpose:** Gets size of `int` type in bytes
- **Usage:** Needed for binary file writing (know how many bytes to write)
- **Example:** Usually 4 bytes on 64-bit systems

**Line 979:** `c_size = sizeof(char);`
- **Purpose:** Gets size of `char` type (usually 1 byte)

**Line 980:** `d_size = sizeof(double);`
- **Purpose:** Gets size of `double` type (usually 8 bytes)

**Line 982:** `n = NumberOfNodes();`
- **Purpose:** Gets total number of wake nodes for this trailing vortex
- **`NumberOfNodes()`:** Member function that returns the count

**Line 984:** `if ( TimeAccurate_ ) n = MIN( Time_ + 1, NumberOfNodes());`
- **Purpose:** For time-accurate analysis, limits number of nodes written
- **`TimeAccurate_`:** Flag indicating time-accurate (unsteady) analysis
- **`Time_`:** Current time step
- **`MIN(...)`:** Macro that returns the smaller value
- **Why:** In unsteady analysis, only write nodes up to current time step

**Line 986:** `if ( TE_Node_Region_Is_Concave_ ) n = 1;`
- **Purpose:** Special case: if trailing edge region is concave, only write 1 node
- **`TE_Node_Region_Is_Concave_`:** Flag for concave trailing edge regions
- **Why:** Concave regions have simplified wake representation

**Line 988:** `fwrite(&(TE_Node_), i_size, 1, adb_file);`
- **Purpose:** Writes trailing edge node ID to binary ADB file
- **`fwrite`:** Binary file write function
- **`&(TE_Node_)`:** Address of the node ID (pointer to data)
- **`i_size`:** Size of each element (int size)
- **`1`:** Number of elements to write
- **`adb_file`:** Binary file to write to
- **Note:** This is for ADB file only, not text file

**Line 990:** `s = double (SoverB_);`
- **Purpose:** Converts span location to double
- **`SoverB_`:** Span location normalized by span (0 to 1)
- **`double (...)`:** Explicit type cast

**Line 992:** `fwrite(&(s), d_size, 1, adb_file);`
- **Purpose:** Writes span location to binary ADB file

**Line 994:** `fwrite(&(n), i_size, 1, adb_file);`
- **Purpose:** Writes number of nodes to binary ADB file
- **Why:** Reader needs to know how many coordinate triplets follow

**Line 996:** `for ( i = 1 ; i <= n ; i++ ) {`
- **Purpose:** Loop through each wake node
- **`i = 1`:** Start at index 1 (this codebase uses 1-based indexing)
- **`i <= n`:** Continue until all nodes processed
- **`i++`:** Increment counter after each iteration

**Line 998:** `x = double (NodeList_[i].x());`
- **Purpose:** Gets X coordinate of wake node `i`
- **`NodeList_[i]`:** Array of wake nodes (VSP_NODE objects)
- **`.x()`:** Member function that returns X coordinate
- **`double (...)`:** Converts to double precision

**Line 999:** `y = double (NodeList_[i].y());`
- **Purpose:** Gets Y coordinate (same pattern as X)

**Line 1000:** `z = double (NodeList_[i].z());`
- **Purpose:** Gets Z coordinate (same pattern as X)

**Lines 1002-1004:** Binary file writes (existing code)
- **Purpose:** Write coordinates to binary ADB file
- **Not explained in detail** (existing functionality)

**Line 1007:** `if ( WakeTextFile != nullptr ) {`
- **Purpose:** Check if text file is open before writing
- **`!= nullptr`:** Checks that file pointer is not null
- **Why check:** File might not be open (error opening, or not writing wake data)
- **Safety:** Prevents crash if file wasn't opened

**Line 1008:** `fprintf( WakeTextFile, "% .6e % .6e % .6e\n", x, y, z );`
- **Purpose:** Writes wake point coordinates to text file
- **`fprintf`:** Formatted print to file
- **Format string:** `"% .6e % .6e % .6e\n"`
  - `% .6e`: Scientific notation, 6 decimal places, space for sign
  - Three values: x, y, z coordinates
  - `\n`: Newline (one point per line)
- **Arguments:** `x, y, z` - the coordinate values
- **Example output:** ` 1.234567e+00  2.345678e+00  3.456789e+00`
- **Format:** Space-separated values (not comma-separated, despite header saying "x,y,z")

**Note:** The header says "x,y,z" but the actual data is space-separated. This is a minor inconsistency but doesn't affect parsing.

---

## Part 4: Closing the File
**File:** `VSP_Solver.C` (Lines 3469-3477)

```cpp
// ======= My modification ===========
// ---- close our text wake dump ----
if ( WakeTextFile )
{
    fclose( WakeTextFile );
    WakeTextFile = nullptr;
}
// Reset flag only at end of sweep (Case <= 0)
if ( Case <= 0 )
{
    WakeTextFileFirstWrite = 1;  // Reset for next sweep
}
// -----------------------------------
```

### Line-by-Line Explanation:

**Line 3471:** `if ( WakeTextFile )`
- **Purpose:** Check if file is open before closing
- **Why:** File might already be closed or never opened

**Line 3473:** `fclose( WakeTextFile );`
- **Purpose:** Closes the file and flushes any remaining data
- **`fclose`:** Standard C function to close a file
- **Why important:** Ensures all data is written to disk

**Line 3474:** `WakeTextFile = nullptr;`
- **Purpose:** Sets pointer to null after closing
- **Why:** Prevents accidentally using a closed file pointer
- **Safety:** Null pointer is safe to check, closed file pointer is not

**Line 3476:** `if ( Case <= 0 )`
- **Purpose:** Check if this is the end of a sweep
- **`Case`:** Case number (0 or negative typically means last case)
- **Why:** Only reset flag at end of sweep, not after each case

**Line 3478:** `WakeTextFileFirstWrite = 1;`
- **Purpose:** Reset flag for next sweep
- **Effect:** Next sweep will start fresh (truncate file)

---

## File Format Summary

The output file will look like this:

```
# Alpha:  3.000000e+00  Mach:  1.000000e-01  Beta:  0.000000e+00
x,y,z
 1.234567e+00  2.345678e+00  3.456789e+00
 1.234568e+00  2.345679e+00  3.456790e+00
 1.234569e+00  2.345680e+00  3.456791e+00
...
# Alpha:  4.000000e+00  Mach:  1.000000e-01  Beta:  0.000000e+00
x,y,z
 1.234567e+00  2.345678e+00  3.456789e+00
 1.234568e+00  2.345679e+00  3.456790e+00
...
```

**Note:** The data values are space-separated (not comma-separated), despite the "x,y,z" header.

---

## Key Concepts

1. **Static Variables:** Persist across function calls, useful for file handles
2. **File Modes:** "w" truncates, "a" appends
3. **1-based Indexing:** This codebase uses arrays starting at index 1
4. **Binary vs Text:** ADB file is binary (fwrite), text file is ASCII (fprintf)
5. **Null Checks:** Always check file pointers before use to prevent crashes

