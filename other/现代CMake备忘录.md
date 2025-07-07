### add_subdirectory
用于将子目录(包含独立`CMakeLists.txt`文件)添加倒当前构建系统中
基本语法：
```cmake
add_subdirectory(source_dir [binary_dir] [EXCLUDE_FROM_ALL])
```
- **`source_dir`**（必需）：  
    子目录路径（相对当前 `CMakeLists.txt` 或绝对路径），需包含子项目的 `CMakeLists.txt` 文件。
- **`binary_dir`**（可选）：  
    子项目的构建输出目录（默认为 `${CMAKE_CURRENT_BINARY_DIR}/source_dir`）。
- **`EXCLUDE_FROM_ALL`**（可选）：  
    若设置，子目录的目标不会默认构建（需显式指定构建）。

