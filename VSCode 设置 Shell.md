
## VS Developer Shell

用户设置 settings.json 通过 vswhere.exe 来查找

```json
"terminal.integrated.profiles.windows": {
	
	"VS Developer Powershell": {
		"path": "C:\\Windows\\System32\\WindowsPowerShell\\v1.0\\powershell.exe",
		"args": [
			"-NoExit",
			"-Command",
			"& { $vs = & 'C:\\Program Files (x86)\\Microsoft Visual Studio\\Installer\\vswhere.exe' -latest -products * -requires Microsoft.Component.MSBuild -property installationPath; Import-Module (Join-Path $vs 'Common7\\Tools\\Microsoft.VisualStudio.DevShell.dll'); Enter-VsDevShell -VsInstallPath $vs -DevCmdArguments '-arch=x64 -host_arch=x64' }"
		],
		"icon": "terminal-powershell"
	}
},
```
