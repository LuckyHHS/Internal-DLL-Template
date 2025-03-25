# Internal-DLL-Template

A basic DLL template, works for x64 mainly, but should on x32/x86. Inject into process to run it.

To build call ``gcc -c main.cp``, then ``gcc -shared -o mydll.dll main.o -lstdc++``

``` cpp
#include <Windows.h>
#include <iostream>
#include <vector>
#include <Psapi.h> // To get process info


// Resolves a multi-level pointer address using a base pointer and a chain of offsets
uintptr_t GetPointerAddress(uintptr_t baseAddress, const std::vector<uintptr_t>& offsets) {
    uintptr_t address = baseAddress;
    try {
        for (size_t i = 0; i < offsets.size(); ++i) {
            address = *(uintptr_t*)address;
            address += offsets[i];
        }
    } catch (...) {
        printf("Access violation occurred while resolving pointer address\n");
        return 0;
    }
    return address;
}

// Retrieves the base address of the main module in the target process
uintptr_t GetBaseAddress(HANDLE hProcess) {
    if (hProcess == NULL)
        return NULL; // No access to the process

    HMODULE moduleArray[1024];
    DWORD neededBytes = 0;

    if (!EnumProcessModules(hProcess, moduleArray, sizeof(moduleArray), &neededBytes))
        return NULL; // Failed to enumerate modules

    TCHAR moduleName[MAX_PATH];
    if (!GetModuleFileNameEx(hProcess, moduleArray[0], moduleName, sizeof(moduleName) / sizeof(TCHAR)))
        return NULL; // Failed to retrieve module name

    return (uintptr_t)moduleArray[0]; // Return the base address of the main module
}

// Main thread function for the DLL
DWORD WINAPI MainThread(HMODULE hModule) {
    // Allocate a console for debugging
    AllocConsole();
    FILE* consoleStream;
    freopen_s(&consoleStream, "CONOUT$", "w", stdout);

    printf("DLL injected successfully!\n");

    // Retrieve the base address of the current process
    HANDLE currentProcess = GetCurrentProcess();
    uintptr_t moduleBase = GetBaseAddress(currentProcess);


    // Resolve pointer chain to the target addresses. Use double since the ammo is a double pointer.
    double* ammoAddress = (double*)GetPointerAddress(moduleBase + 0x00DCC520, {0x0, 0x48, 0x10, 0xF0, 0x0});


    // While loop to run code.
    while (!GetAsyncKeyState(VK_BACK)) {
        if (ammoAddress) {
            *ammoAddress = 99999999999.0; // Update ammo value
        } else {
            printf("Failed to dereference address\n");
        }
        Sleep(100);
    }

    // Cleanup
    fclose(consoleStream);
    FreeConsole();
    FreeLibraryAndExitThread(hModule, 0);
    return 0;
}

// Entry point for the DLL
BOOL APIENTRY DllMain(HMODULE hModule, DWORD ulReasonForCall, LPVOID lpReserved) {
    switch (ulReasonForCall) {
        case DLL_PROCESS_ATTACH:
            // Create a new thread for the main function
            CloseHandle(CreateThread(nullptr, 0, (LPTHREAD_START_ROUTINE)MainThread, hModule, 0, nullptr));
            break;
        case DLL_PROCESS_DETACH:
            break;
    }
    return TRUE;
}
```
