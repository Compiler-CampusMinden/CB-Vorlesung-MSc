---
archetype: lecture-cg
title: "Error-Recovery"
author: "Carsten Gips (HSBI)"
weight: 5
readings:
  - key: "Parr2010"
    comment: "Kapitel 2 und 3"
  - key: "Parr2014"
  - key: "Levine2009"
    comment: "Kapitel 7 und 8"
tldr: |
  Umgang mit Fehlern ist im Compiler sehr wichtig: Falscher Code darf nicht in ein ausführbares Programm
  umgewandelt werden oder ausgeführt werden, gleichzeitig erwarten Nutzer zielführende Fehlermeldungen
  und auch das Erkennen von möglichst mehreren Fehlern in einem Lauf.

  Auf der Ebene des Parsers kann man Fehler in Bezug auf die Grammatik erkennen. Typische Strategien sind
  das Entfernen von Token aus dem Eingabestrom, bis wieder ein Token erscheint, welches die weitere Abarbeitung
  der aktuellen Regel erlaubt ("Synchronisierung"). Dies sind oft Zeilenenden, ein Semikolon oder eine schließende
  geschweifte Klammer. Dieses recht einfache, aber grobe Vorgehen kann verfeinert werden, indem man versucht,
  überschüssige Token zu entfernen oder fehlender Token zu ersetzen. In ANTLR wird beispielsweise maximal ein
  fehlendes Token virtuell "ersetzt" bzw. max. ein überschüssiges Token entfernt, damit man den restlichen Code
  weiter parsen kann. Wenn mehr als ein Token fehlt oder zu viel ist, geht ANTLR in einen "Panic Mode" und
  entfernt so lange Token aus dem Eingabestrom, bis das aktuelle Token in einem *Resynchronization Set* enthalten
  ist. Die Bildung dieser Menge erinnert an die Regeln zum Bilden der *FOLLOW*-Mengen, ist aber an den Kontext
  der "aufgerufenen" Parser-Regeln gebunden. Zusätzlich gibt es weitere komplexere Strategien zum Behandeln von
  Fehlern in Schleifen sowie zur Vermeidung von Endlos-Fehlerbehebungsschleifen ("Fail-Save"). Diese Form der
  Behandlung stellt einen Kompromiss zwischen Aufwand (auch Zeit) und Nutzen dar.

  Zusätzlich kann man in der Grammatik bereits typische Fehler (vergessene Klammern oder Typos wie Dreher bei
  Schlüsselwörtern) schon über "Fehlerproduktionen" vorwegnehmen. Das bedeutet, dass man eine Regel formuliert,
  die diesen typischen Tippfehler akzeptiert (und korrigiert), aber zusätzlich eine Warnung generiert. Es muss
  dann aber jeweils entschieden werden, ob der entsprechende Quellcode in ein ausführbares Programm übersetzt
  werden darf.
outcomes:
  - k2: "Varianten der Fehler bei Parsern"
  - k2: "Fehlerbehandlung bei LL-Parsern: *single token deletion*, *single token insertion*, *sync-and-return*"
  - k2: "Berechnung und Anwendung des *Resynchronization Sets*"
  - k3: "Ändern der Fehlermeldungen bei ANTLR"
  - k3: "Eigene Errorhandler in ANTLR-Parser bauen und einbinden"
  - k3: "Nutzen von speziellen Fehler-Alternativen in Grammatiken"
assignments:
  - topic: sheet01
    name: "VL Error-Recovery"
youtube:
  - link: "https://youtu.be/9sFdI7pYMQs"
fhmedia:
  - link: "https://www.hsbi.de/medienportal/m/eabf5e829fbcd6be866e20b169989c8fef4fd10f13704999d0c1b531d15d4255975cd4490ac127156874d8334d6cade0ece8e2d15f2f2d34cb965a0c1697eade"
    name: "VL Error-Recovery"
---


## Fehler beim Parsen

![](images/bc_xml-parsing-error.png)

[Quelle: Vorlesung "Einführung in die Programmierung mit Skriptsprachen" by [BC George](mailto:bc.george@hsbi.de) ([CC BY-SA 4.0](http://creativecommons.org/licenses/by-sa/4.0/))]{.origin}

::: notes
*   Compiler ist ein schnelles Mittel zum Finden von (syntaktischen) Fehlern
*   Wichtige Eigenschaften:
    *   Reproduzierbare Ergebnisse
    *   Aussagekräftige Fehlermeldungen
    *   Nach Erkennen eines Fehlers: (vorläufige) Korrektur und Parsen des restlichen Codes
        => weitere Fehler anzeigen.
        Problem: Bis wohin "gobbeln", d.h. was als Synchronisationspunkt nehmen? Semikolon?
    *   Syntaktisch fehlerhafte Programme dürfen nicht in die Zielsprache übersetzt werden!

_Anmerkung_: Die folgenden Inhalte beziehen sich auf die Fehlerbehandlung in ANTLR (v4).
:::


## Typische Fehler beim Parsing

```antlr
grammar VarDef;

alt   : stmt | stmt2 ;
stmt  : 'int' ID ';' ;
stmt2 : 'int' ID '=' ID ';'  ;
```

::: slides
[Grammatik: [VarDef.g4](https://github.com/Compiler-CampusMinden/CB-Vorlesung-Master/blob/master/markdown/parsing/src/VarDef.g4), Input-Beispiele: [VarDef.txt](https://github.com/Compiler-CampusMinden/CB-Vorlesung-Master/blob/master/markdown/parsing/src/VarDef.txt)]{.bsp}
:::


::::::::: notes
*Anmerkung*: Die nachfolgenden Fehler werden am Beispiel der Grammatik
`[VarDef.g4](src/VarDef.g4)`{=markdown} und ANTLR demonstriert.

### Lexikalische Fehler

Eingabe: `int x1;` (Startregel `stmt`)

Fehlermeldung: `token recognition error at: '1'`

Die ist ein Fehler aus dem Lexer, wenn beim Erkennen eines Tokens ein komplett
unbekanntes Zeichen auftritt.

### Ein extra Token

Eingabe: `int x y;` (Startregel `stmt`)

Fehlermeldung: `extraneous input 'y' expecting ';'`

Wenn nur ein Token zu viel ist, dann kann der von ANTLR generierte Parser eine
passende Fehlermeldung ausgeben.

### Mehrere extra Token

Eingabe: `int x y z;` (Startregel `stmt`)

Fehlermeldung: `mismatched input 'y' expecting ';'`

Wenn dagegen mehr als ein Token zu viel ist, dann gibt der von ANTLR generierte
Parser eine generische Fehlermeldung aus.

### Fehlendes Token

Eingabe: `int ;` (Startregel `stmt`)

Fehlermeldung: `missing ID at ';'`

Ein anderer typischer Fehler sind fehlende Token, die kann der Parser analog zu
überzähligen Token erkennen und ausgeben.

### Fehlendes Token am Entscheidungspunkt

Eingabe: `int ;` (Startregel `alt`)

Fehlermeldung: `no viable alternative at input 'int;'`

Hier fehlt ein Token, aber an einer Stelle, wo sich der Parser zwischen zwei
Alternativen (Sub-Regeln) entscheiden muss.
:::::::::


## Überblick Recovery bei Parser-Fehlern

![](images/recovery.png){width="80%"}

::: notes
*   Fehler im Lexer (hier nicht weiter betrachtet):
    *   Aktuelles Zeichen passt zu keinem Token: Entfernen oder Hinzufügen
        von Zeichen (plus Rückmeldung an den Parser)
    *   Spezielle Token, die typische fehlerhafte Zeichenketten als Token
        erkennen (mit Weiterverarbeitung im Parser)

*   Fehler im Parser:
    *   Token passt nicht: Token entfernen oder ein Dummy-Token erzeugen
    *   Panic-Mode: Entferne Token bis zu einem Synchronisationspunkt.
        Problem: Dabei nicht zu weit zu springen!
    *   Spezielle Fehlerproduktionen: Spezielle Regeln in der Grammatik,
        die typische Fehler matchen.
:::


## Skizze: Generierte Parser-Regeln (ANTLR)

```antlr
stmt  : 'int' ID ';' ;
```

\bigskip

```python
def stmt():
    try: match("int"); match(ID); match(";")
    catch (RecognitionException re):
        _errHandler.reportError(self)               # let's report it
        _errHandler.recover(self)                   # Panic-Mode

def match(x):
    if lookahead == x: consume()
    else: _errHandler.recoverInline(self)           # Inline-Mode
}
```

::: notes
Der im Parser registrierte ErrorHandler erzeugt in der Methode
`reportError()` eine geeignete Meldung und gibt sie an den Parser
über dessen Methode `notifyErrorListeners()` weiter.

Die eigentliche Fehlerbehandlung findet in der Methode `recover()`
bzw. `recoverInline()` des ErrorHandlers statt.
:::


## Inline-Recovery bei Token-Mismatch (Skizze)

```python
def recoverInline(parser):
    # SINGLE TOKEN DELETION
    if singleTokenDeletion(parser):
        return getMatchedSymbol(parser)

    # SINGLE TOKEN INSERTION
    if singleTokenInsertion(parser):
        return getMissingSymbol(parser)

    # that didn't work, throw a new exception
    throw new InputMismatchException(parser)
}
```

::: notes
Die Klasse `InputMismatchException` drückt aus, dass das aktuelle Token nicht
zur Erwartung des Parsers passt. Deshalb wird diese Exception am Ende von
`recoverInline()` geworfen. Die Klasse `RecognitionException`, die in den
Parserregeln wie `stmt` gefangen wird, ist die gemeinsame Oberklasse aller
Parser-Exceptions.


Liste der wichtigsten Exceptions (nach
[github.com/antlr/antlr4/blob/master/doc/parser-rules.md](https://github.com/antlr/antlr4/blob/master/doc/parser-rules.md)):

| Exception                   | Beschreibung                                                                           |
|:----------------------------|:---------------------------------------------------------------------------------------|
| `RecognitionException`      | Basisklasse für alle Parser-Exceptions                                                 |
| `NoViableAltException`      | Parser konnte sich nicht für (mind.) einen Pfad entscheiden angesichts des Tokenstroms |
| `LexerNoViableAltException` | Lexer-Pendant zu `NoViableAltException`                                                |
| `InputMismatchException`    | Das aktuelle Token ist nicht das, was der Parser erwartet                              |

:::


## Panic Mode: Sync-and-Return (Skizze)

```python
def rule():
    try: ... rule-body ...
    catch (RecognitionException re):
        _errHandler.reportError(self)       # let's report it
        _errHandler.recover(self)           # Panic-Mode
}
```

\bigskip

=> Entferne solange Token, bis aktuelles Token im "*Resynchronization Set*"


## Einsatz des "*Resynchronization Set*"

::: notes
*   **Following Set**: Menge der Token, die direkt auf eine Regel-Referenz folgen,
    ohne dass die aktuelle Regel/Alternative verlassen wird
*   **Resynchronization Set**: Vereinigung der *Following Sets* für alle Regeln im
    aktuellen Aufruf-Stack

[Quelle: nach [@Parr2014, pp. 161-163]]{.origin}
:::

\bigskip

```antlr
stmt : 'if' expr ':' stmt           // Following Set für "expr": {':'}
     | 'while' '(' expr ')' stmt ;  // Following Set für "expr": {')'}
expr : term '+' INT ;               // Following Set für "term": {'+'}
```

\bigskip

*   Eingabe: `if :`
*   Aufruf-Stack nach Bearbeitung von `if`: `[stmt, expr, term]`
*   **Resynchronization Set**: `{'+', ':'}`

[[Hinweis: *FOLLOW* $\ne$ *Following*]{.bsp}]{.slides}


::: notes
### Hinweis: *FOLLOW* != *Following*

**FOLLOW** ist die Menge aller Token, die auf eine Regel folgen können

*   `FOLLOW(term) = {'+'}`
*   `FOLLOW(expr) = {':', ')'}`


**Following** ist dagegen **abhängig vom aktuellen Kontext**!

*   Stack: `[stmt, expr, term]` => *Resynchronization Set*: `{'+', ':'}`
:::


::: notes
### Beispiele Resynchronisation im Panic Mode (ANTLR)

**Hinweis**: Die Regel `term` ist in obigem Beispiel nicht weiter detailliert. Hier wird
angenommen, dass das aktuelle Token `':'` nicht passt.

*   Eingabe: `if :`
    *   In Regel `term`: Token `':'` passt nicht
        *   `consume()`, bis aktuelles Token in *Resynchronization Set*: `{'+', ':'}`
            (d.h. hier bleibt `':'` das aktuelle Token)
        *   Rückkehr zu Regel `expr`
    *   In Regel `expr`: Token `':'` passt nicht
        *   `consume()`, bis aktuelles Token in *Resynchronization Set*: `{':'}`
            (d.h. hier bleibt `':'` das aktuelle Token)
        *   Rückkehr zu Regel `stmt`
    *   In Regel `stmt`: Token `':'` passt jetzt
        *   Abschluss des Parsing (mit Fehlermeldung)

*   Eingabe: `if x + 42 ))):`
    *   In Regel `stmt`: Token `')'` passt nicht
        *   `consume()`, bis aktuelles Token in *Resynchronization Set*: `{':'}`  (d.h.
            hier werden alle `')'` entfernt)
    *   In Regel `stmt`: Token `':'` passt jetzt
        *   Abschluss des Parsing (mit Fehlermeldung)
:::


::: notes
## Anmerkungen Fehlerbehandlung in Sub-Regeln

Bei Sub-Regeln (d.h. eine Regel enthält Alternativen) oder Schleifenkonstrukten
(d.h. eine Regel enthält `(...)*` oder `(...)+`) geht ANTLR etwas anders vor.

1.  Start einer Sub-Regel/Alternative: Versuch einer *single token deletion*

    ```python
    # am Anfang einer Alternative oder Schleife
    _errHandler.sync(self)
    ```

2.  Schleifenkonstrukte: `(...)*` oder `(...)+`

    Versuche, in der Schleife zu bleiben! Im Fehlerfall `consume()` bis

    *   Weitere Iteration der Schleife erkannt
    *   Token, welches der Schleife folgt, erkannt
    *   Token im *Resynchronization Set* des aktuellen Aufruf-Stacks

    Anmerkung: Im Prinzip entspricht dies dem *Panic Mode*, der Unterschied liegt
    darin, bis wohin der Parser nach der Recovery in einer Funktion/Methode (Regel)
    zurückspringt. D.h. wenn es verschiedene Möglichkeiten gibt, haben diese die
    obige Priorisierung.

3.  Fail-Save

    Um Endlos-Schleifen durch die Schritte (1) bzw. (2) zu vermeiden, löst der Parser
    beim zweiten Versuch, die selbe Parser-Stelle und Input-Position zu bearbeiten
    (also bei bereits aktivem Fehler), einen "*Fail-Safe*" aus. Der Parser konsumiert
    dann ein Token und fährt dann mit der Recovery fort.

Zu Details zur Fehlerbehandlung durch ANTLR vergleiche [@Parr2014, S. 170 ff.].
:::


::::::::: notes
## Ändern der Fehlerbehandlungs-Strategie in ANTLR

### Ändern der Fehlerbehandlungs-Strategie (global)

![](images/handler.png)

Sie überschreiben die Klasse `DefaultErrorStrategy` und müssen die oben gezeigten Methoden
`recover()` und `recoverInline()`aufrufen. Die eigene Fehlerbehandlung setzen Sie über die
Methode `setErrorHandler` des Parsers.

### Ändern der Fehlerbehandlungs-Strategie (lokal)

```antlr
r : ...
  ;
  catch[RecognitionException e] { throw e; }
```

Es lassen sich auch andere bzw. mehrere Exceptions fangen. Der `catch`-Block ersetzt den
Default-`catch`-Block der generierten Methode. Das bedeutet, dass sich der geänderte Modus
nur für die eine Regel auswirkt.

### Ändern der Fehler-Meldungen

![](images/listener.png)

Für einen eigenen Listener leitet man sinnvollerweise von `BaseErrorListener` ab und
überschreibt die leere Implementierung von `syntaxError()`.

Damit die Fehlermeldungen nicht mehrfach ausgegeben werden, entfernt man zunächst alle
Listener und fügt dann den eigenen hinzu, bevor man den Parser startet.
:::::::::



## Fehlerproduktionen

::: notes
Häufig vorkommende Fehler kann man bereits in der Grammatik berücksichtigen.
Dadurch kommt es nicht zu einem Parser-Error mit Recovery-Mechanismus, sondern
der Fehler wird über eine entsprechende Alternative in der Grammatik korrigiert.

Es bietet sich an, in diesem Fall eine entsprechende Ausgabe zu tätigen. Dies
wird in der folgenden Grammatik über eingebettete Aktionen erledigt.
:::

```antlr
stmt : 'int' ID ';'
     : 'int' ID             {notifyErrorListeners("Missing ';'");}
     : 'int' ID ';' ';'     {notifyErrorListeners("Too many ';'");}
     ;
```

::: notes
Der aus der Grammatik generierte Parser leitet von der Basisklasse `Parser`
ab. Dort wird eine Methode `notifyErrorListeners()` implementiert, die man
mit Hilfe von in die Grammatik eingebetteten Aktionen aufrufen kann (Vorgriff
auf `["Syntaxgesteuerte Interpreter"]({{<ref "/backend/interpretation/syntaxdriven" >}})`{=markdown}).
Letztlich steht im generierten Parser in der generierten Methode `stmt()` an
der passenden Stelle ein Aufruf `notifyErrorListeners(Too many ';'");` ...
:::


::: notes
## Anmerkung: Nicht eindeutige Grammatiken

```antlr
stat: expr ';' | ID '+' ID ';' ;
expr: ID '+' ID | INT ;
```

=> Was passiert bei der Eingabe: `a+b` ??! Welche Regel/Alternative soll
jetzt matchen, d.h. welcher AST soll am Ende erzeugt werden?!

Nicht eindeutige Grammatiken führen in ANTLR **nicht** zu einer Fehlermeldung,
da nicht der Nutzer mit seiner Eingabe Schuld ist, sondern das Problem
in der Grammatik selbst steckt.

Während des Debuggings von Grammatiken lohnt es sich aber, diese
Warnungen zu aktivieren. Dies kann entweder mit der Option "`-diagnostics`"
beim Aufruf des `grun`-Tools geschehen oder über das Setzen des
`DiagnosticErrorListener` aus der ANTLR-Runtime als ErrorListener.
:::


## Wrap-Up

*   Fehler bei `match()`: *single token deletion* oder *single token insertion*

*   ANTLR: Panic Mode: *sync-and-return* bis Token in *Resynchronization Set*
    *   Sonderbehandlung bei Start von Sub-Regeln und in Schleifen
    *   Fail-Save zur Vermeidung von Endlosschleifen

*   Fehler-Alternativen in Grammatik einbauen






<!-- DO NOT REMOVE - THIS IS A LAST SLIDE TO INDICATE THE LICENSE AND POSSIBLE EXCEPTIONS (IMAGES, ...). -->
::: slides
## LICENSE
![](https://licensebuttons.net/l/by-sa/4.0/88x31.png)

Unless otherwise noted, this work is licensed under CC BY-SA 4.0.
:::
