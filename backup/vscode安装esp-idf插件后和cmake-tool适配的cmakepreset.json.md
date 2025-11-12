```json
{
    "version": 6,
    "cmakeMinimumRequired": {
        "major": 3,
        "minor": 22,
        "patch": 0
    },
    "configurePresets": [
        {
            "name": "base",
            "displayName": "Basic Config",
            "description": "Basic build using Ninja generator",
            "generator": "Ninja",
            "hidden": true,
            "binaryDir": "${sourceDir}/build_win/${presetName}",
            "cacheVariables": {
                "CMAKE_INSTALL_PREFIX": "${sourceDir}/install_win/${presetName}"
            }
        },
        {
            "name": "Debug",
            "cacheVariables": {
                "CMAKE_BUILD_TYPE": "Debug"
            },
            "hidden": true
        },
        {
            "name": "ESP32-S3",
            "hidden": true,
            "cacheVariables": {
                "CMAKE_CXX_COMPILER": "xtensa-esp32s3-elf-g++.exe",
                "CMAKE_C_COMPILER": "xtensa-esp32s3-elf-gcc.exe",
                "CMAKE_ASM_COMPILER": "xtensa-esp32s3-elf-gcc.exe"

            },
            "environment": {
                "PYTHON":"C:/Users/<your-name>/.espressif/python_env/idf5.5_py3.11_env/Scripts/python.exe",
                "PYTHON_HOME": "C:/Users/<your-name>/.espressif/python_env/idf5.5_py3.11_env",
                "PYTHON_EXECUTABLE": "C:/Users/<your-name>/.espressif/python_env/idf5.5_py3.11_env/Scripts/python.exe",
                "IDF_PATH": "C:/Users/<your-name>/esp/v5.5.1/esp-idf",
                "PATH": "C:/Users/<your-name>/.espressif/tools/xtensa-esp-elf/esp-14.2.0_20241119/xtensa-esp-elf/bin;$penv{path}"
            }
        },
        {
            "name": "x64-Debug",
            "description": "ESP32-S3 for x64 (Debug)",
            "inherits": ["base", "Debug", "ESP32-S3"]
        }
    ],
    "buildPresets": [
        {
            "name": "x64-Debug-Build",
            "configurePreset": "x64-Debug",
            "jobs": 10,
            "verbose": true
        }
    ]
}

```
这里的编译仅仅作为cmake-tool代码提示工具，尽管和esp-idf插件的结果基本一致，但是由于要在esp32上烧录和适配，因此请尽量使用esp插件。

另外，作为一个尝试，尝试解决C-compiler不能编译simple program的问题的配置如下，但是这样貌似不能使用python导入所有的path。所以以add_subdirectory的形式导入所有examples失败了（
```json
{
    "version": 6,
    "cmakeMinimumRequired": {
        "major": 3,
        "minor": 22,
        "patch": 0
    },
    "configurePresets": [
        {
            "name": "base",
            "displayName": "Basic Config",
            "description": "Basic build using Ninja generator",
            "generator": "Ninja",
            "hidden": true,
            "binaryDir": "${sourceDir}/build_win/${presetName}",
            "cacheVariables": {
                "CMAKE_INSTALL_PREFIX": "${sourceDir}/install_win/${presetName}"
            }
        },
        {
            "name": "Debug",
            "cacheVariables": {
                "CMAKE_BUILD_TYPE": "Debug"
            },
            "hidden": true
        },
        {
            "name": "ESP32-S3",
            "hidden": true,
            "cacheVariables": {
                "CMAKE_CXX_COMPILER": "xtensa-esp32s3-elf-g++.exe",
                "CMAKE_C_COMPILER": "xtensa-esp32s3-elf-gcc.exe",
                "CMAKE_ASM_COMPILER": "xtensa-esp32s3-elf-gcc.exe",
                "CMAKE_LINKER": "xtensa-esp32s3-elf-ld.exe",
                "CMAKE_SYSTEM_NAME": "Generic",
                "CMAKE_SYSTEM_PROCESSOR": "xtensa"

            },
            "environment": {
                "PYTHON":"C:/Users/<your-name>/.espressif/python_env/idf5.5_py3.11_env/Scripts/python.exe",
                "PYTHON_HOME": "C:/Users/<your-name>/.espressif/python_env/idf5.5_py3.11_env",
                "PYTHON_EXECUTABLE": "C:/Users/<your-name>/.espressif/python_env/idf5.5_py3.11_env/Scripts/python.exe",
                "IDF_PATH": "C:/Users/<your-name>/esp/v5.5.1/esp-idf",
                "PATH": "C:/Users/<your-name>/.espressif/tools/xtensa-esp-elf/esp-14.2.0_20241119/xtensa-esp-elf/bin;$penv{path}"
            }
        },
        {
            "name": "x64-Debug",
            "description": "ESP32-S3 for x64 (Debug)",
            "inherits": ["base", "Debug", "ESP32-S3"]
        }
    ],
    "buildPresets": [
        {
            "name": "x64-Debug-Build",
            "configurePreset": "x64-Debug",
            "jobs": 10,
            "verbose": true
        }
    ]
}


```