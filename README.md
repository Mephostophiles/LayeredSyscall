# LayeredSyscall

Generating legitimate call stack frame along with indirect syscalls by abusing Vectored Exception Handling (VEH) to bypass User-Land EDR hooks in Windows.

Accompanying blog post can be found here: *link to blog here*

## Usage

Include the files, [FuncWrappers.h](https://github.com/WKL-Sec/LayeredSyscall/blob/main/FuncWrappers.h) and [FuncWrappers.cpp](https://github.com/WKL-Sec/LayeredSyscall/blob/main/FuncWrappers.cpp) within your source code.

Make sure to initialize and destroy the hooks using the calls `IntializeHooks()` and `DestroyHooks()` at the beginning and end of your main code.

To let the tool perform the evasion add the prefix "wrp" to the start of the syscall name to be used. For eg.,

`NtCreateUserProcess() -> wrpNtCreateUserProcess()`


Below is a demo code to show how the tool can be used, full code can be found [here](https://github.com/WKL-Sec/LayeredSyscall/blob/main/demo.cpp).
```cpp
int main(int argc, char* argv[]) {

	printf("[*] Program Started\n");
	IntializeHooks();

	WCHAR image_path[] = L"\\??\\C:\\Windows\\System32\\calc.exe";
	UNICODE_STRING NtImagePath;
	RtlInitUnicodeString(&NtImagePath, (PWSTR)L"\\??\\C:\\Windows\\System32\\calc.exe");

	// Create the process parameters
	PRTL_USER_PROCESS_PARAMETERS ProcessParameters = NULL;
	RtlCreateProcessParametersEx(&ProcessParameters, &NtImagePath, NULL, NULL, NULL, NULL, NULL, NULL, NULL, NULL, RTL_USER_PROCESS_PARAMETERS_NORMALIZED);

	// Initialize the PS_CREATE_INFO structure
	PS_CREATE_INFO CreateInfo = { 0 };
	CreateInfo.Size = sizeof(CreateInfo);
	CreateInfo.State = PsCreateInitialState;

	// Initialize the PS_ATTRIBUTE_LIST structure
	PPS_ATTRIBUTE_LIST AttributeList = (PS_ATTRIBUTE_LIST*)RtlAllocateHeap(RtlProcessHeap(), HEAP_ZERO_MEMORY, sizeof(PS_ATTRIBUTE));
	AttributeList->TotalLength = sizeof(PS_ATTRIBUTE_LIST) - sizeof(PS_ATTRIBUTE);
	AttributeList->Attributes[0].Attribute = PS_ATTRIBUTE_IMAGE_NAME;
	AttributeList->Attributes[0].Size = NtImagePath.Length;
	AttributeList->Attributes[0].Value = (ULONG_PTR)NtImagePath.Buffer;

	HANDLE hProcess, hThread = NULL;

	BOOL success = FALSE;
	LPVOID remote_base_addr = NULL;
	CONTEXT thread_context = { 0 };

	// create the target process in a suspended state so we can modify its memory and the context of its main thread
	if (wrpNtCreateUserProcess(&hProcess, &hThread, PROCESS_ALL_ACCESS, THREAD_ALL_ACCESS, NULL, NULL,
		NULL, NULL, ProcessParameters, &CreateInfo, AttributeList))
	{
		printf("CreateProcessA() Failed with error: %d\n", GetLastError());
		return -1;
	}

	DestroyHooks();
	printf("[*] Program Ended\n");
	return 1;
}
```

## Results

### Call stack analysis

Performing indirect syscall shows no legitimate call stack
![image](https://github.com/user-attachments/assets/c360d8a3-9ab8-4736-bd45-3aa371ad386e)

Legitimate call stack after usage of the tool
![image](https://github.com/user-attachments/assets/f5d79970-e699-4408-b658-97db512fd90c)


### Testing against an EDR

Before using the tool with Process Hollowing technique against Sophos EDR,
![image](https://github.com/user-attachments/assets/b9a65ad9-42f5-4c94-b8ec-6fcb47bbe74e)

After using the tool with Process Hollowing technique against Sophos EDR,
![image](https://github.com/user-attachments/assets/bc42702e-9795-4d96-8cf0-3f9e8f287fbe)

## Potential Detections

As of now, detections against this technique would require one to check for maliciously registered exception handlers within a particular program. Other detections could also include flagging anomalous stack behavior by implementing a heuristic against known call stack produced by Windows APIs.
