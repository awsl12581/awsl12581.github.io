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
            "binaryDir": "${sourceDir}/build/${presetName}",
            "cacheVariables": {
                "CMAKE_INSTALL_PREFIX": "${sourceDir}/install/${presetName}"
            }
        },
        {
            "name": "x64",
            "architecture": {
                "value": "x64",
                "strategy": "external"
            },
            "cacheVariables": {
                "DIRECTX_ARCH": "x64"
            },
            "hidden": true
        },
        {
            "name": "Debug",
            "cacheVariables": {
                "CMAKE_BUILD_TYPE": "Debug"
            },
            "hidden": true
        },
        {
            "name": "Release",
            "cacheVariables": {
                "CMAKE_BUILD_TYPE": "Release",
                "CMAKE_INTERPROCEDURAL_OPTIMIZATION": true
            },
            "hidden": true
        },
        {
            "name": "MSVC",
            "hidden": true,
            "cacheVariables": {
                "CMAKE_CXX_COMPILER": "cl.exe"
            },
            "toolset": {
                "value": "v143,host=x64,version=14.44.35207",
                "strategy": "external"
            }
        },
        {
            "name": "MSVC2026",
            "hidden": true,
            "cacheVariables": {
                "CMAKE_POLICY_VERSION_MINIMUM": "3.5",
                "CMAKE_CXX_COMPILER": "cl.exe",
                "CMAKE_CUDA_FLAGS": "--allow-unsupported-compiler"
            },
            "toolset": {
                "value": "v145,host=x64,version=14.50.35717",
                "strategy": "external"
            }
        },
        {
            "name": "x64-Debug",
            "description": "MSVC for x64 (Debug)",
            "inherits": ["base", "x64", "Debug", "MSVC"]
        },
        {
            "name": "x64-Release",
            "description": "MSVC for x64 (Release)",
            "inherits": ["base", "x64", "Release", "MSVC"]
        },
        {
            "name": "x64-Debug-2026",
            "description": "MSVC 2026 for x64 (Debug)",
            "inherits": ["base", "x64", "Debug", "MSVC2026"]
        },
        {
            "name": "x64-Release-2026",
            "description": "MSVC 2026 for x64 (Release)",
            "inherits": ["base", "x64", "Release", "MSVC2026"]
        }
    ],
    "buildPresets": [
        {
            "name": "x64-Debug-Build",
            "configurePreset": "x64-Debug",
            "jobs": 10,
            "verbose": true
        },
        {
            "name": "x64-Release-Build",
            "configurePreset": "x64-Release",
            "jobs": 10,
            "verbose": false
        },
        {
            "name": "x64-Debug-2026-Build",
            "configurePreset": "x64-Debug-2026",
            "jobs": 10,
            "verbose": true
        },
        {
            "name": "x64-Release-2026-Build",
            "configurePreset": "x64-Release-2026",
            "jobs": 10,
            "verbose": false
        }
    ]
}
```

记得修改``version=xxxx.xxxx.xxxx....``版本号才会起作用，另外CUDA暂时不支持vs2026，所以要添加``"CMAKE_CUDA_FLAGS": "--allow-unsupported-compiler"``