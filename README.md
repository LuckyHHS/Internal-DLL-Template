# Internal-DLL-Template

A basic DLL template, works for x64 mainly, but should on x32/x86. Inject into process to run it.


Use MSYS and add both Bin folders to the PATH environment variable. ``C:\msys64\mingw32\bin`` and ``C:\msys64\ucrt64\bin``
To install MSYS 32 bit version, call ``pacman -S mingw-w64-i686-gcc`` in the msys terminal.

**64 Bit:** Call ``gcc -c main.cpp`` then ``gcc -shared -o mydll.dll main.o -lstdc++ ``

**32 Bit:** Call ``i686-w64-mingw32-gcc -c main.cpp -o main.o`` then ``i686-w64-mingw32-gcc -shared -o mydll.dll main.o -lstdc++``

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

// Main thread function for the DLL
DWORD WINAPI MainThread(HMODULE hModule) {
    // Allocate a console for debugging
    AllocConsole();
    FILE* consoleStream;
    freopen_s(&consoleStream, "CONOUT$", "w", stdout);

    printf("DLL injected successfully!\n");

    // Retrieve the base address of the current process
    uintptr_t moduleBase = (uintptr_t )GetModuleHandle("UnityPlayer.dll");

    printf("Got module base adress and current process!");

    // Resolve pointer chain to the target addresses. CHANGE TYPE OF POINTER TO YOUR TYPE OF VALUE, HEALTH IS FLOAT IN THIS CASE.
    float* ammoAddress = (float*)GetPointerAddress(moduleBase + 0x01CA9428, {0x9F8, 0x6C8, 0x28, 0x38, 0xD8, 0x38, 0x194});

    printf("%f", (*ammoAddress));

    // While loop to run code.
    while (!GetAsyncKeyState(VK_BACK)) {
        if (ammoAddress) {
            *ammoAddress = 100.0; // Update ammo value
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
