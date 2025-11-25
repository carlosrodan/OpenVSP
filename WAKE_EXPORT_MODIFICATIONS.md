# Wake Points Text Export Modification

## Overview

This modification adds the ability to export wake point coordinates to a text file in addition to the existing binary ADB format. This enables easier parsing with Python for visualization and analysis purposes.

## Files Modified

### 1. `src/vsp_aero/Solver/Vortex_Trail.H`
- **Line 465**: Updated `WriteToFile()` function declaration to accept a second parameter `FILE *WakeTextFile`

```cpp
void WriteToFile(FILE *adb_file, FILE *WakeTextFile);
```

### 2. `src/vsp_aero/Solver/Vortex_Trail.C`
- **Line 970**: Updated `WriteToFile()` function implementation to accept the text file parameter
- **Lines 1007-1009**: Added code to write wake point coordinates to the text file

```cpp
// ====== My modification =========
if ( WakeTextFile != nullptr ) {
    fprintf( WakeTextFile, "% .6e % .6e % .6e\n", x, y, z );
}
// ================================
```

### 3. `src/vsp_aero/Solver/VSP_Solver.C`
- **Line 14**: Added static file pointer for the wake text file
- **Lines 32575-32584**: Open the text file before writing wake data
- **Line 32594**: Updated function call to pass the text file pointer
- **Lines 3469-3476**: Close the text file after all data is written

## How It Works

1. **File Creation**: When `WriteOutAerothermalDatabaseSolution()` is called, a text file named `{FileName}_wake.txt` is created in write mode.

2. **Data Export**: For each trailing vortex, wake point coordinates (x, y, z) are written to both:
   - The binary ADB file (existing functionality)
   - The text file (new functionality)

3. **File Format**: The text file contains one wake point per line with space-separated coordinates:
   ```
   x1 y1 z1
   x2 y2 z2
   x3 y3 z3
   ...
   ```

4. **File Closure**: The text file is closed after all wake data has been written, typically at the end of the `Solve()` function.

## Output File

- **Filename**: `{FileName}_wake.txt` (where `{FileName}` is the base name of the ADB file)
- **Location**: Same directory as the ADB file
- **Format**: ASCII text with floating-point coordinates in scientific notation

## Usage

The text file is automatically generated whenever wake data is written to the ADB file (non-time-accurate cases). No additional configuration is required. The file can be parsed with Python using standard text file reading methods.

## Notes

- The text file export only occurs for non-time-accurate cases (same condition as ADB file writing)
- If the text file cannot be opened, an error message is printed but execution continues
- The modification maintains backward compatibility with existing ADB file format

