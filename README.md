# NBL-2-Emulator
Der Versuch einen NBL2 Emulator zu bauen und die Fallstricke.

Ich wollte einen NBL-2 Emulator bauen, da dieser von Teltonika und deren Trackern unterstützt wird. Auf Basis des ESP32 und KI Unterstützung fast ein Kinderspiel.
Netronix und Teltonika bieten gute Doku dazu, wenn wenn sie sich gegenseitig widersprechen. 
Die Emulation funktionierte in den NBLTools perfekt. Alles wurde richtg angezeigt, nur der Teltonika Tracker weigerte sich. 
Hier konnte ich die Daten nur auslesen, wenn ich einen Advanced Sensor konfigurierte.
Auf Antwort vom Support wartet man ewig und per Mail wird man sehr auf einen Sale gedrängt.

Also wurde mit hilfe von KI die Firmware reversed Engineerd. Beeindruckend was LLMs hier leisten, und bei welchen kleinen Problemen sie stolpern.
Jedenfalls wurde da Problem gefunden.

TLDR;
Entgegen der Dokumentation wird keine Bytelänge von 21B angenommen, sondern von 20B. 
Code um 1 Byte gekürzt und es läuft -.- Lustigeweise wird die Emulation nun als NBL-4 in den NBLTools erkannt :)



