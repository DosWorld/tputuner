
           TPUTUNER - Ein Code-Optimierer for Turbo Pascal
       =======================================================

Downloaded from: http://www.phost.de/~stefan/turbo.html

Funktion
---------

  TPUTUNER optimiert eine Turbo Pascal Unit (.tpu), um bessere
  Performance und vor allem auch geringere Codegr��e zu erreichen. 
  Dazu wird die Unit disassembliert, optimiert, und wieder
  assembliert.

  TPUTUNER unterst�tzt Turbo Pascal 6.0 (bzw. TPW 1.5) und Borland
  Pascal 7.0; zwischen beiden Modi wird zur Compilierzeit
  unterschieden.


Aufruf
-------

  tputuner unit.tpu [ziel.tpu] [optionen]

  Die Zieldatei kann von der Unitdatei verschieden sein, um das
  Original zu erhalten. Folgende Optionen sind m�glich:

  -a, --data-flow-analysis
  -j, --reduce-jump-chains
  -p, --peephole-optimizations
  -u, --remove-unused-code
  -e, --early-jump
  -l, --late-jump
  -r, --reg-alloc
  -m, --sort-moves
  -c, --combine-strings
  -g, --cse
       Legen die Art der Optimierung fest. Standardm��ig sind alle
       Optionen bis auf -r, -c und -g eingeschaltet. Um eine Option
       auszuschalten, verwenden Sie Gro�buchstaben (also `-A')
       bzw. bei der langen Option `--no-' (also `--no-early-jump').
       Siehe auch die Beschreibung der einzelnen Optionen weiter
       unten.

  -d, --debug-dump
       Aktiviert die Erzeugung von Debug-Dumps, siehe unten.

  -n, --names
       Wenn aktiv (Standard) liest TPUTUNER die Symboltabellen aus um
       in den Statusmeldungen die Namen der bearbeiteten Funktionen
       anzeigen zu k�nnen. Wenn dies st�rt oder problematisch wird,
       kann es mit `-N' / `--no-names' abgeschaltet werden.

  -s, --size
       Optimiert auf Codegr��e, nicht Geschwindigkeit. Damit werden
       teilweise Befehle erzeugt, die langsamer sind (i.a. mehr
       Speicherzugriffe), aber weniger Code ben�tigen.

  -3, --386
       Erlaubt die Nutzung von 386-Befehlen. Standardm��ig liest und
       schreibt TPUTUNER 286er-Code.


Statusmeldungen
----------------

  W�hrend des Ablaufs werden Meldungen des folgenden Formates
  ausgegeben:

  - code block #248... 1! 2 3 r1  --> 47 bytes saved (new size = 1247)
                 \      \      \       \__ gesparter Code in diesem Block
                  \      \      \__ Reassembler-Passes
                   \      \__ Optimierer-Passes
                    \__ Codeblock-Nummer

  + Codeblock-Nummern: die erste implementierte Prozedur erh�lt Nummer
    0, dann folgt Nummer 8, 10, 18 (hex) usw., unabh�ngig von der
    Deklarations-Reihenfolge im INTERFACE-Teil.

  + Optimierer-Passes: `!' markieren Passes, in denen eine
    Code-Verbesserung auftrat. Nach 2 Passes mit identischem Ergebnis
    wird die Optimierung beendet.

  + Reassembler-Passes: meistens nur einer, mehrere k�nnen auftreten,
    wenn sich durch den ersten Pass noch Adressen verschoben haben, so
    da� Spr�nge k�rzer codiert werden konnten.


Fehlermeldungen
----------------

  Alle diese Meldungen f�hren dazu, da� der entsprechende Block nicht
  angefa�t wird. Damit soll Code erkannt werden, den TPUTUNER kaputt
  machen w�rde (I/O, Interrupt-Aufrufe, Daten im Codesegment, usw.)

  `Unrecognized opcode xx'
  `Direct interrupt access'
  `FPU is not supported' und �hnliche
    TPUTUNER kann nicht alle Befehle disassemblieren. Nicht erkannt
    werden alle 386+-Befehle, `test', Flag-Befehle, Port-I/O,
    String-Befehle.

  `Warning: code block contains bad relocation entries'
    Relozierungseintr�ge sind Stellen, an denen der Linker eine
    Adresse nachtragen mu�. TPUTUNER alle F�lle behandeln, die Turbo
    Pascal erzeugt, einige Inline-Assembler-Konstruktionen m�ssen
    jedoch zur�ckgewiesen werden, z.B. `@foo: mov ax, offset @foo'.

  `Unrecognized relocation item at...'
    Relozierungen d�rfen nur in 16-bit-Operanden auftreten. Eine
    Relozierung woanders ist ein sicheres Zeichen f�r Daten im
    Codesegment.


Inline-Assembler
-----------------

  ... wird fr�hlich mit-optimiert. Dies ist erw�nscht. TPUTUNER h�tte
  sowieso keine M�glichkeit, Pascal- und Assembler-Code zu
  unterscheiden. Damit werden u.a. Spr�nge gek�rzt (`jmp finish'
  ... `finish: Exit;') und Speichertransfers eliminiert (`a :=
  Random(10); asm mov ax,a ...'). Kritische Assembler-Routinen outen
  sich hoffentlich durch nicht unterst�tzte Befehle im Code.

  Inline-Funktionen werden ebenfalls optimiert. Beispiel:
    function nachfolger(i:integer):integer; inline($58/$40);
    ...
    j:=nachfolger(3);
  wird von Turbo Pascal zu
     push 3
     pop  ax     ; $58
     inc  ax     ; $40
     mov  [j],ax
  TPUTUNER optimiert die push/pop-Kombination weg zu
     mov  ax,3

  Eingebundene .OBJ-Dateien werden i.A. nicht optimiert, da eine
  .OBJ-Datei von Turbo Pascal einen einzigen Code-Block mit mehreren
  Eintrittspunkten zugewiesen bekommt. In Assembler gelten auch die
  �blichen Sichtbarkeitsregeln nicht, so da� man fr�hlich zwischen
  Prozeduren hin- und herspringen kann, was TPUTUNER nicht handhaben
  kann.


Optimierungen
--------------

* Datenflu�-Analyse (--data-flow-analysis)

  Eliminiert �berfl�ssige Befehle, wenn die Ergebnisse, die sie
  erzeugen, bereits ermittelt wurden. Das sind vor allem Ladebefehle,
  aber auch Logikbefehle, wenn ein Operand 0 ist. Wahrscheinlich w�re
  der richtige Name `reload optimisation', aber `Datenflu�-Analyse'
  klingt einfach cooler ;-)

  + Beispiel:
        p^.a := 1;
        p^.b := 2;
    wird zu
        les(2)    di, [ss:bp+6]
        mov(2)    [es:di+0], 1
        les(2)    di, [ss:bp+6]
        mov(2)    [es:di+2], 2
    TPUTUNER streicht das zweite LES.

  + Fehlerbeispiel: Aliasing
        p := @a;
        a := b + 1;
        inc(p^);
        b := a + 1;
    wird zu:
        lea(2)    ax, [ss:bp+8]            ; p := @a
        mov(2)    dx, ss
        mov(2)    [ss:bp+fffffffc], ax
        mov(2)    [ss:bp+fffffffe], dx
        mov(2)    ax, [ss:bp+6]            ; a := b + 1
        inc(2)    ax
        mov(2)    [ss:bp+8], ax
        les(2)    di, [ss:bp+fffffffc]     ; inc(p^)
        inc(2)    [es:di+0]
        mov(2)    ax, [ss:bp+8]        **  ; b := a + 1
        inc(2)    ax
        mov(2)    [ss:bp+6], ax

    Dieser Code inkrementiert b dreimal. Der optimierte Code
    inkrementiert b nur zweimal, da die mit ** markierte Zeile als
    unn�tz verworfen wird, "schlie�lich enth�lt AX bereits den Wert von
    [ss:bp+8]".

  + Fehlerbeispiel: Parameter in Registern
                mov     si,offset foo
                mov     di,si
                call    bar  { Quelle in SI, Ziel in DI }
    wird optimiert zu
                mov     di, offset foo
                call    bar
    TPUTUNER nimmt an, da� SI nur f�r die Initialisierung von DI
    geladen wurde, und ist sicher, alles �ber SI zu wissen.


* Sprungketten entfernen (--reduce-jump-chains)

  Wenn das Ziel eines Sprungs ein weiterer Sprungbefehl ist, wird
  direkt zum Endziel gesprungen. Das kommt h�ufig bei Konstruktionen wie
         IF foo THEN BEGIN
           IF bar THEN qux ELSE frob
         END ELSE fred;


* `Guckloch-Optimierungen' (--peephole-optimizations)

  Ersetzt komplexe Befehle durch einfachere. Das ist eine Art `Suchen
  und Ersetzen' und sollte mit TP-Code sicher sein. Hier werden die
  Befehle einzeln in einem Umfeld von nicht mehr als 2 oder 3 anderen
  Befehlen betrachtet, daher ``Guckloch''. Ein Beispiel ist hier
                cmp(2)  ax, 0
  wird zu
                or(2)   ax, ax
  egal, in welchem Kontext.

  Au�erdem ersetzt diese Optimierung einige Aufrufe in die Unit SYSTEM
  durch Assembler-Instruktionen (siehe tpufmt.h), und modifiziert
  gelegentlich den Initialisierungscode einer FOR-Schleife.


* Unbenutzten Code entfernen (--remove-unused-code)

  Entfernt unreferenzierten Code, der unter anderem durch
  Sprungketten-Entfernung entsteht. Sollte immer sicher sein, au�er
  bei Daten im Code...


* Spr�nge vorziehen (--early-jump)

  Hier werden Spr�nge eher durchgef�hrt, wenn am Ziel des Sprungs
  dieselben Befehle durchgef�hrt werden wie vor dem Sprung. Das tritt
  bei vielen Bedingungen zu, besonders wenn Strings oder Longints
  beteiligt sind. TPUTUNER erkennt zwei Bl�cke auch dann als gleich,
  wenn sie sich in einem Befehl unterscheiden, den TPUTUNER
  korrigieren kann. Wenn diese Optimierung verwendet wird, darf die
  Entfernung unbenutzten Codes (`-U') nicht abgeschaltet werden.

  + Beispiel:
                cmp     [xxx], 1        IF xxx=1 THEN
                jne     skip
                push    10                func(10)
                call    func
                jmp     done            ELSE
        skip:   push    9                 func(9);
                call    func
        done:
    kann verk�rzt werden zu
                cmp     [xxx], 1
                jne     skip
                push    10
                jmp     done
        skip:   push    9
        done:   call    func


* Spr�nge verz�gern (--late-jump)

  Diese Optimierung funktioniert �hnlich wie `--early-jump', hier
  werden jedoch Spr�nge verz�gert. Im Gegensatz zu `--early-jump' wird
  hier jedoch nicht immer Code gespart, der sich ergebende Code ist
  aber u.U. besser parallelisierbar.

  + Beispiel:
                cmp     [xxx],1         IF xxx=1 THEN
                jne     skip
                push    10                Fred(10)
                call    Fred
                jmp     done            ELSE
        skip:   push    10                Barney(10);
                call    Barney
        done:
    kann verk�rzt werden zu
                cmp     [xxx],1
                push    10
                je      skip
                call    Fred
                jmp     done
        skip:   call    Barney
        done:


* Optimierung auf Gr��e (--size)

  Die meisten Optimierungen bringen sowohl einen Geschwindigkeits-
  gewinn, als auch reduzierte Programmgr��e. Manche �nderungen sind
  jedoch mit einer Geschwindigkeitseinbu�e verbunden. TPUTUNER wird
  diese nur dann durchf�hren, wenn Sie die --size-Option angegeben
  haben. Beispiel:
                mov     [foo], ss
                mov     es, [foo]
  wird ohne diese Option unver�ndert gelassen, mit --size wird diese
  Sequenz zu
                mov     [foo], ss
                push    ss
                pop     es
  was langsamer ist (mehr Speichertransfers), aber 2 Bytes k�rzer. Den
  `mov [foo],ss' kann TPUTUNER nicht entfernen, da der Wert in [foo]
  evtl. sp�ter noch gebraucht wird.

  Allerdings kann auch ohne `-s' der Code langsamer werden, z.B. bei
  ung�nstigen Konstellationen mit `--reg-alloc'.


* Register-Allokierung (--reg-alloc)

  Diese Optimierung veranla�t TPUTUNER dazu, h�ufig ben�tigte Werte oder
  Adressen in Registern abzulegen. Das k�nnen Konstanten, lokale
  Variablen und Adressen sein. In kleinen Prozeduren ohne
  Unterprogrammaufrufe werden lokale Variablen u.U. komplett in
  Register gelegt.

  Diese Optimierung ist standardm��ig deaktiviert, da sie sehr
  zeitaufwendig ist. Da sie stellenweise Ergebnisse der DFA r�ckg�ngig
  macht (und dabei *immer* bessere lies kleinere Ergebnisse liefert),
  kann sie nur angewendet werden, wenn die DFA keine �nderungen mehr
  bringt.

  + Beispiel:
                var1 := 7;
                var2 := 7;
                var3 := 7;
    wird von Turbo Pascal zu
                mov     [var1],7
                mov     [var2],7
                mov     [var3],7
    und von TPUTUNER zu
                mov     cx,7
                mov     [var1],cx
                mov     [var2],cx
                mov     [var3],cx

    Letztere Sequenz ist k�rzer (von der Codegr��e her) und nicht
    unbedingt langsamer (da weniger Speichertransfers und in anderen
    Konstellationen als im Beispiel u.U. besser parallelisierbar).


* Ladebefehle sortieren (--sort-moves)

  Sortiert `mov'-Befehle. Dadurch k�nnen diese besser zu 16- und
  32-bit Operationen zusammengefasst werden. Allerdings werden damit
  Lock-Variablen ad absurdum gef�hrt:
        lock := TRUE;
        critical1 := 1;
        critical2 := 42;
        lock := FALSE;
  sortiert, je nach Adre�lage, die beiden Zuweisungen an lock an den
  Anfang oder das Ende.

  Diese Optimierung ist Bestandteil der Peephole-Optimierung, also mu�
  letztere eingeschaltet sein, um --sort-moves zu verwenden.


* Common Subexpression Elimination (--cse)

  Ungef�hr das, was die CSE in "richtigen" Compilern macht: wiederholende
  Codeschnipsel weglassen. Wenn TPUTUNER eine Codefolge der Form
        <codeA>
        <codeB>
        <codeA>
  findet, wobei <codeB> das Ergebnis von <codeA> bestehen l��t, kann
  das zweite <codeA> weggelassen werden. Z.B. wird `p^.q^.x := p^.q^.y;'
  von Turbo Pascal zu
        les     di, [p]             \  codeA
        les     di, [es:di+10]      /
        mov     ax, [es:di]         >  codeB
        les     di, [p]             \
        les     di, [es:di+10]      /  codeA
        mov     [es:di+2], ax
  Die zweite Adre�rechnung kann weggelassen werden.

  Diese Optimierung ist standardm��ig ausgeschaltet, weil
  + sie langsam ist (die Suche nach gleichen Codest�cken ist mindestens
    O(n^2))
  + hier Zeiger-Aliasing-Probleme besonders deutlich werden. TPUTUNER hat
    eine ultrasimple Aliasing-Erkennung f�r diese Optimierung (und gar
    keine Aliasing-Erkennung f�r den Rest!). Wie leicht man diese aushebeln
    kann, verbleibt zu ermitteln: wenn sowohl <codeA> als auch <codeB>
    auf eine Speicherzelle [es:di+10] schreiben, wird Aliasing angenommen;
    da Turbo Pascal keine Zeigerarithmetik kann, sollten Probleme relativ
    selten sein...


* String-Kombinierung (--combine-strings)

  (noch nicht fertig) Turbo Pascal fa�t gleiche Stringkonstanten nur
  innerhalb einer Funktion zusammen. Diese Optimierung tut dies f�r
  eine komplette Unit. Allerdings mu� sie dazu sehr konservativ
  vorgehen, um nicht andere Konstanten (float) zu zerst�ren.

  NOTE 10/Okt/2000: Vermutlich werde ich diese Optimierung jedoch
  wieder entfernen. Sie zu schreiben ist ziemlich bescheuert, und es
  bringt nicht soooo viel.


* Nutzung von 386er-Befehlen (--386)

  Erzeugt 386er-Befehle:
  + jcc j16
  + setcc r/m8
  + push i32, push m32
  + mov r/m32, i32

  Die 32-bit-Befehle tauchen in den Debuglistings nicht auf, sie
  werden als zwei 16-bit-Befehle mit derselben Adresse gelistet:
        98:  push(2)   [es:di+ffffffea]
        98:  push(2)   [es:di+ffffffe8]
  entspricht einem `push(4) [es:di+ffffffe8]'.

  TPUTUNER kann so bearbeitete Units nicht wieder einlesen!


* Was TPUTUNER nicht tut...

  + Vorausberechnung von Konstanten: das sollte der Turbo Pascal
    Compiler im Rahmen seiner M�glichkeiten getan haben.

  + Loop Unrolling (Schleifen aufrollen)

  + Function Inlining (Ideen f�r einen globalen(!) Inliner bestehen
    aber -- das kann nicht mal gcc ;-)

  + EXE-Dateien optimieren. Bei TPUs steht an den Konstanten noch
    dran, welche eine Adresse ist und welche nicht -- bei EXEn nicht.

  + Aliasing-Erkennung: TPUTUNER geht davon aus, da� kein Wert �ber
    zwei Zugriffspfade erreichbar ist, also z.B. �ber einen Zeiger und
    als feste Adresse. Quasi wird angenommen, alle Zeiger w�ren mit
    dem C-Modifier `restrict' definiert; keine Variable ist `volatile'
    (der Turbo-Pascal-Compiler nimmt genau das Gegenteil an).


Ergebnisse
-----------

  TPUTUNER kann die meisten Units um 2-10% verk�rzen. Ich halte das
  f�r ein ganz ansehnliches Ergebnis.

  Beispiel-Ergebnisse:

  + PCC (ca. Version 0.99.5):
    - ca. 40 KLOC + 5 KLOC Bibliothekscode (nicht optimiert)
    - Normaler Turbo Pascal Code:
       cc.exe  112144
       cc.ovr  352649
             = 464793
    - Optimierung mit `tputuner -3sE', Version ca. 11/1998:
       cc.exe  110256
       cc.ovr  341475
             = 451731 (13062 = 2.8% gespart)

  + PCC (v1.0.6.3-debug):
    - ca. 70 KLOC
    - Normaler Turbo Pascal Code:
       cc.exe  244544
       cc.ovr  642800
             = 887344
    - Optimierung mit `tputuner -3rg', Version 10/2000:
       cc.exe  230672
       cc.ovr  583492
             = 814164 (73180 = 8.2% gespart)

  (Bei so gro�en interaktiven Programmen sollte man wohl vorrangig auf
  Gr��e optimieren -- Antwortzeiten sind eigentlich nie ein Problem)

  Die wichtigsten Optimierungen sind Datenflu�analyse (`-a') und
  Spr�nge vorziehen (`-e') sowie evtl. CSE (`-g'). Die anderen
  Optimierungen alleine bringen nur marginale Verbesserungen. Alle
  zusammen angewendet k�nnen jedoch das Ergebnis verbessern: so kann
  die Sprungkettenverfolgung einige Spr�nge unreferenziert machen, so
  da� sie beim Entfernen unbenutzten Codes Speicherplatz freigeben.


Optimierung unterst�tzen
-------------------------

  Wenn's auf jedes Byte ankommt...

  + Einfache Anweisungen / WITH: TPUTUNER verwaltet gr��tenteils nur
    eine Indirektionsebene. Die CSE kann mehr, erwischt aber auch nicht
    alles.
  + Zuweisungsketten / Wert direkt nach Ermittlung benutzen, damit
    Registerwerte wiederverwendet werden k�nnen.
  + Variablen zusammenfassen: Variablen, die oft gemeinsam genutzt
    werden, in den selben VAR-Block direkt beieinander packen (->
    mehrere Variablen k�nnen mit einem `mov' verwendet werden).
  + Kurze, h�ufig benutzte Variablen zuerst deklarieren, vor den
    gro�en (-> 8-bit Displacements).
  + Variablen, die h�ufig im Paar als Parameter �bergeben werden,
    zusammen definieren: statische Variablen/Record-Elemente als
    `VAR y, x:INTEGER', lokale als `VAR x, y:INTEGER' (-> `foo(x,y)'
    kann diese als einen LONGINT �bergeben bekommen).


Debuglistings
--------------

  Die Debuglistings (--debug-dump) haben die Dateinamen
  `blockX.passY' (`blockX.pY' unter DOS), wobei X die Blocknummer (hex)
  und Y die Pass-Nummer (dez) ist. Es werden nur die Optimizer-Passes
  protokolliert, nicht die Reassembler-Passes, da letztere effektiv
  nichts �ndern.

  Das Format der Dumps ist ungef�hr so:

  1a:  mov(2)    cx, 3
            \__ Operandgr��e in Klammern, wichtig bei Befehlen ohne
                Register-Operanden
  1f:  callf     <un=13,rt=30,rb=30,ro=0>
          \                  \___ Verweis auf einen Relocation-Eintrag
           \___ `call far', im Gegensatz zu `call near'
  24:  mov(2)    [ss:bp+6], ax
  2e:  jcc`c'    <label:38>
         \            \_ Ziel = Label (derzeit an Offset 38hex)
          \_ Bedingter Sprung mit Condition Code (jcc`c' == jl, Opcode 7Ch),
             auch bei setcc (setcc`c' == setl, Opcode 0Fh 9Ch).
  38:  label`1'  <38>
   \          \_ Bei Labels: Referenzz�hler (hex) = wieviel Spr�nge
    \            verweisen auf dieses Label
     \_ Adresse (Hex)

  Manchmal erhalten Befehle einen zus�tzlichen Operanden untergejubelt
  (z.B. `xor(2)  ax, ax, 0'), der f�r interne Abl�ufe in TPUTUNER
  wichtig ist.

  TPUTUNER erzeugt auch im Nicht-386er-Modus 386er-Befehle in den
  Debug-Listings. Diese werden in der Ausgabe vom Assembler wieder
  auseinandergenommen.

  Relocation-Eintr�ge werden von TPUTUNER nicht weiter analysiert, bis
  auf die Annahme, da� der Wert `ro=' zur Berechnung des Abstandes im
  Speicher von zwei sonst identischen Eintr�gen benutzt werden kann,
  und das bestimmte Relocations Systemaufrufe der Unit SYSTEM bezeichnen
  (siehe tpufmt.h): long_mul, long_sqr und sload.


Compilieren
------------

  Ich compiliere TPUTUNER �blicherweise unter Linux mit gcc-2.8.1 oder
  neuer mit dem beliegenden Makefile. Unter DOS, mit DJGPP 2.02+
  compiliert TPUTUNER "out-of-the-box" mit GNU make.


Autor, Legalesisch
-------------------

  Stefan Reuther, <Streu@gmx.de>

  Kopieren dieses Programmes ist ausdr�cklich erlaubt. Wenn Sie dieses
  Programm f�r mehr als blo� mal einen kurzen Test einsetzen, senden Sie
  doch bitte eine Mail mit Erfahrungsbericht oder �hnlichem.

  KEINE GARANTIE: DER EINSATZ VON tputuner ERFOLGT AUF EIGENE GEFAHR.
  F�R DURCH tputuner ENTSTANDENE SCH�DEN LEHNE ICH JEGLICHE HAFTUNG AB.
  WENN SIE SICH MIT tputuner EIN PROGRAMM KAPUTTMACHEN, M�SSEN SIE F�R
  DEN SCHADEN SELBST AUFKOMMEN. WENN SIE DIESES RISIKO NICHT AUF SICH
  NEHMEN WOLLEN, D�RFEN SIE tputuner NICHT EINSETZEN.

  Ich verwende tputuner im t�glichen Einsatz mit PCC (70000+ Zeilen),
  "kritische" Fehler fallen mir also meistens auf. Vielleicht verwenden
  Sie aber ander Code-Muster als ich, so das tputuner auf die Nase f�llt.
  Oder der Fehler ist so subtil, da� er kaum auff�llt. Ich hatte einmal
  einen Fall, da� in meiner grafischen Oberfl�che die Beschriftungen der
  Buttons um 1 Pixel verschoben waren -- schuld war tputuner. Trotzdem
  scheint tputuner relativ stabil zu sein.


-eof-
