## New VSCode extensions for security analists

In the past year I have been developing some extensions for Visual Studio Code that simplifys my daily work.
With my past as a developer, I'm still atached to VSCode for opening/editing text files, CSV files, JSONs and any kind of text files that includes Indicators of Compromise or are interesting from a CyberSecurity point of view.

One of this daily task include checking if a office document with macros is malicious or not. In some clients is common to use office documents as a scripting alternative for tasks like "listing" online servers doing "pings" and writing the result in a file. Most of the time because office is alredy installed and python modules are difficult to install without a local admin account.

For that reason, I developed the extension **[Macro Lab](https://marketplace.visualstudio.com/items?itemName=SecSamDev.macro-lab)**, which allows me to dig into the guts of an office document without having to install anything on the system, just load an extension in VSCode.

![Flow of working with Macro Lab](https://raw.githubusercontent.com/SecSamDev/vscode-office-macro/main/doc/HowToUse.gif)

Most of the code that interacts with the Office binary format is pure JavaScript that I developed for an older private project called Maldoker. 

One of these days Maldoker will see the light of day, but for the moment, I'll just say that it's a lightweight sandbox for Office macro documents that can give you IOCs in less than a second for each macro file. In the previous version of Maldoker with only one processor it was possible to analyze 10 documents per second, the new version reaches 50 in the worst case.

Returning to the main topic, the other daily task was Integrate security devices in SIEMs such as Firewalls, Antivirus, EDR, IDS / IPS. All these devices generate a lot of logs and I need to use Regexes all day and I couldn't use Regex online services like reex101 because I needed to pre-anonymize customer logs. Then came **[Grok Pattern](https://marketplace.visualstudio.com/items?itemName=SecSamDev.grok)**, and I was able to check which records were not being processed and fix them, version the Regex with Git and export Regex for use in Java or Python applications within a string (with double quotes: "). 

![Exporting regex](https://raw.githubusercontent.com/SecSamDev/grok-vscode/master/doc/grok-export.gif)


And finally, one of the last tools to be born was **[VirusTotal](https://marketplace.visualstudio.com/items?itemName=SecSamDev.virustotal)**, to incorporate a functionality similar to [Munin] (https: //github.com/Neo23x0/munin) but within an IDE.
With this I could have a simple Munin. The next step is to be able to share a common database using a shared folder with other analysts on my team. 

![Analyze list of IOCs](https://raw.githubusercontent.com/SecSamDev/vscode-virustotal/main/doc/ImportIOClist.jpg)