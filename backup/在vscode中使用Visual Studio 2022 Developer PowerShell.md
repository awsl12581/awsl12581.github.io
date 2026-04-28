```json
{
    "python.testing.pytestArgs": [
        "test"
    ],
    "python.testing.unittestEnabled": false,
    "python.testing.pytestEnabled": true,
    "[python]": {
        "editor.defaultFormatter": "ms-python.black-formatter",
        "editor.formatOnSave": true
    },
    "python-envs.defaultEnvManager": "ms-python.python:conda",
    "python-envs.defaultPackageManager": "ms-python.python:conda",
    "python-envs.pythonProjects": [],
    "python.envFile": "${workspaceFolder}/.env",
    "terminal.integrated.suggest.enabled": false,
    "terminal.integrated.defaultProfile.windows": "VSDev_PowerShell(2022)",
    "terminal.integrated.profiles.windows": {
        "VSDev_PowerShell(2022)": {
            "args": [
                "-noe",
                "-c",
                "&{Import-Module \"C:\\Program Files\\Microsoft Visual Studio\\2022\\Community\\Common7\\Tools\\Microsoft.VisualStudio.DevShell.dll\"; Enter-VsDevShell 2dab4dc2 -SkipAutomaticLocation -DevCmdArguments \"-arch=x64 -host_arch=x64\"}"
            ],
            "source": "PowerShell",
            "icon": "terminal-powershell"
        },
        "VsDevCmd (2022)": {
            "path": [
                "${env:windir}\\Sysnative\\cmd.exe",
                "${env:windir}\\System32\\cmd.exe"
            ],
            "args": [
                "/k",
                // Path below assumes a VS2022 Community install; 
                // update as appropriate if your IDE installation path
                // is different, or if using the standalone build tools
                "C:/Program Files/Microsoft Visual Studio/2022/Community/Common7/Tools/VsDevCmd.bat",
                "-arch=x64",
                "-host_arch=x64"
            ],
            "overrideName": true,
            "icon": "terminal-cmd"
        },
    },
}
```
请注意修改
Enter-VsDevShell 2dab4dc2
方法: 按照桌面terminal中的setting来确定