
parse(Template):
===============
	» Template [text]
	» Header [text]
	« Tree [list]

		Template.splitAt: “PANEL {…}\n”
			Before » Panel [text]
			Capture » NextHeader [text]
			After » Remainder [text]
	
		Panel.parse: Header » Tree.add:
		Remainder.parse(Template): NextHeader » Tree.join:


parse(Panel):
============
	» Panel [text]
	» Header [text]
	« Tree [list]

		Header.parse: » Tree.metadata.join:
	
		Panel.splitAt: “\n”
			Before » Line [text]
			After » Remainder [text]
	
		Line.parse: » Tree.add:
		Remainder.parse(Panel): » Tree.join:


parse(Header):
=============
	» Header [text]
	« Options [list]

		Header.splitAt: “, ”
			Before » Option [text]
			After » Remainder [text]
	
		Option.parse: » Options.add:
		Remainder.parse(Header): » Options.join:


parse(Option):
=============
	» Option [text]
	« Setting [text]

Create a key-value pair from “key=value” text.

		Option.splitAt: ”=”
			Before » Key [text]
			After » Value [text]
	
		Key=Value » Setting


parse(Line):
===========
	» Line [text]
	« Tree [list]

First parse line options at the end	-- separated like this (after a tab-stop).

		Line.splitAt: “\t-- ”
			Before » Content [text]
			After » Header [text]
	
		Header.parse: » Tree.metadata.join:

Next, parse brackets. No nesting or overlapping.

Step 1: “Be fore [Inn er] After” → {“Be”, “fore”}, “Inn er] After”

		Content.splitAt: “[” or “_” or “**”
			Before » splitWords: Label=“Word” » Tree.join:
			Split » Bracket [text]
			After » Tail [text]

Step 2: {“Be”, “fore”}, “Inn er ] After” → {“Be”, “fore”, Variable=“Inn er”} + {After.parsed}

		Tail.splitAt: Bracket.inverse:
			Before » Inner [text]
			After » Remainder [text]
		
		Inner.parse: Bracket » Tree.join:
		Remainder.parse(Line): » Tree.join:


label(Bracket):
==============
	» Bracket [text]
	« Label [text]

		(Bracket == “[”) ⇒ “Variable”
		(Bracket == “_”) ⇒ “Italics”
		(Bracket == “**”) ⇒ “Bold”


inverse(Bracket):
================
	» Bracket [text]
	« Inverse [text]

		(Bracket == “[”) ⇒ “]”
		(Bracket == “_”) ⇒ “_”
		(Bracket == “**”) ⇒ “**”


parse(Inner):
============
	» Inner [text]
	» Bracket [text]
	« Result [list]

Italics and bold text are split into words; variables aren’t.

		(Bracket == “[”) ⇒ {Bracket.label:=Inner}
		(Bracket == “_”) ⇒ splitWords: {Block=Inner, Label=Bracket.label:}
		(Bracket == “**”) ⇒ splitWords: {Block=Inner, Label=Bracket.label:}


splitWords:
==========
	» Block [text]
	» Label [text]
	« Words [list]

		Block.splitAt: “ ”
			Before » Word [text]
			After » After [text]
	
		Label=Word » Words.add:
		splitWords: {Block=After, Label} » Words.join:
