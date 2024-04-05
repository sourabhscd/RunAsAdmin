The script takes advantage of the fact that NET FILE requires administrator privilege and returns errorlevel 1 if you don't have it. The elevation is achieved by creating a script which re-launches the batch file to obtain privileges. This causes Windows to present the UAC dialog and asks you for the administrator account and password.

I have tested it with Windows 7, 8, 8.1, 10, 11 and with Windows XP - it works fine for all. The advantage is, after the start point you can place anything that requires system administrator privileges, for example, if you intend to re-install and re-run a Windows service for debugging purposes (assumed that mypackage.msi is a service installer package):

msiexec /passive /x mypackage.msi
msiexec /passive /i mypackage.msi
net start myservice

Without this privilege elevating script, UAC would ask you three times for your administrator user and password - now you're asked only once at the beginning, and only if required.

If your script just needs to show an error message and exit if there aren't any administrator privileges instead of auto-elevating, this is even simpler: You can achieve this by adding the following at the beginning of your script:

@ECHO OFF & CLS & ECHO.
NET FILE 1>NUL 2>NUL & IF ERRORLEVEL 1 (ECHO You must right-click and select &
  ECHO "RUN AS ADMINISTRATOR"  to run this batch. Exiting... & ECHO. &
  PAUSE & EXIT /D)
REM ... proceed here with admin rights ...

This way, the user has to right-click and select "Run as administrator". The script will proceed after the REM statement if it detects administrator rights, otherwise exit with an error. If you don't require the PAUSE, just remove it. Important: NET FILE [...] EXIT /D) must be on the same line. It is displayed here in multiple lines for better readability!

On some machines, I've encountered issues, which are solved in the new version above already. One was due to different double quote handling, and the other issue was due to the fact that UAC was disabled (set to lowest level) on a Windows 7 machine, hence the script calls itself again and again.

I have fixed this now by stripping the quotes in the path and re-adding them later, and I've added an extra parameter which is added when the script re-launches with elevated rights.

The double quotes are removed by the following (details are here):

setlocal DisableDelayedExpansion
set "batchPath=%~0"
setlocal EnableDelayedExpansion

You can then access the path by using !batchPath!. It doesn't contain any double quotes, so it is safe to say "!batchPath!" later in the script.

The line

if '%1'=='ELEV' (shift & goto gotPrivileges)

checks if the script has already been called by the VBScript script to elevate rights, hence avoiding endless recursions. It removes the parameter using shift.

Update:

    To avoid having to register the .vbs extension in Windows 10, I have replaced the line
    "%temp%\OEgetPrivileges.vbs"
    by
    "%SystemRoot%\System32\WScript.exe" "%temp%\OEgetPrivileges.vbs"
    in the script above; also added cd /d %~dp0 as suggested by Stephen (separate answer) and by Tomáš Zato (comment) to set script directory as default.

    Now the script honors command line parameters being passed to it. Thanks to jxmallet, TanisDLJ and Peter Mortensen for observations and inspirations.

    According to Artjom B.'s hint, I analyzed it and have replaced SHIFT by SHIFT /1, which preserves the file name for the %0 parameter

    Added del "%temp%\OEgetPrivileges_%batchName%.vbs" to the :gotPrivileges section to clean up (as mlt suggested). Added %batchName% to avoid impact if you run different batches in parallel. Note that you need to use for to be able to take advantage of the advanced string functions, such as %%~nk, which extracts just the filename.

    Optimized script structure, improvements (added variable vbsGetPrivileges which is now referenced everywhere allowing to change the path or name of the file easily, only delete .vbs file if batch needed to be elevated)

    In some cases, a different calling syntax was required for elevation. If the script does not work, check the following parameters:
    set cmdInvoke=0
    set winSysFolder=System32
    Either change the 1st parameter to set cmdInvoke=1 and check if that already fixes the issue. It will add cmd.exe to the script performing the elevation.
    Or try to change the 2nd parameter to winSysFolder=Sysnative, this might help (but is in most cases not required) on 64 bit systems. (ADBailey has reported this). "Sysnative" is only required for launching 64-bit applications from a 32-bit script host (e.g. a Visual Studio build process, or script invocation from another 32-bit application).

    To make it more clear how the parameters are interpreted, I am displaying it now like P1=value1 P2=value2 ... P9=value9. This is especially useful if you need to enclose parameters like paths in double quotes, e.g. "C:\Program Files".

    If you want to debug the VBS script, you can add the //X parameter to WScript.exe as first parameter, as suggested here (it is described for CScript.exe, but works for WScript.exe too).

    Bugfix provided by MiguelAngelo: batchPath is now returned correctly on cmd shell. This little script test.cmd shows the difference, for those interested in the details (run it in cmd.exe, then run it via double click from Windows Explorer):

    @echo off
    setlocal
    set a="%~0"
    set b="%~dpnx0"
    if %a% EQU %b% echo running shell execute
    if not %a% EQU %b% echo running cmd shell
    echo a=%a%, b=%b% 
    pause

Useful links:

    Meaning of special characters in batch file:
    Quotes ("), Bang (!), Caret (^), Ampersand (&), Other special characters

