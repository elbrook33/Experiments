
parse(Template):
=============
## In
* Template [text]

## Out
* Tree [list: “template”]

	Template.splitAt: “PANEL {…}\n”
		Before » Panel [text]
		Capture » Header [text]
		After » Remainder [text]

	Panel.parse: Header » Tree.add:
	Remainder.parse(Template): » Tree.join:


parse(Panel):
==========
## In
* Panel [text]
* Header [text]

## Out
* Tree [list: “panel”]	

	Header.parse: » Tree.metadata.join:

	Panel.splitAt: “\n”
		Before » Line [text]
		After » Remainder [text]

	Line.parse: » Tree.add:
	Remainder.parse(Panel): » Tree.join:


parse(Header):
===========
» Header [text]
« Options [list]

	Header.splitAt: “, ”
		Before » Option [text]
		After » Remainder [text]

	Option.parse: » Options.add:
	Remainder.parse(Header): » Options.join:


parse(Option):
===========
» Option [text]
« Setting [pair]

	Option.splitAt: ”=”
		Before » Key [text]
		After » Value [text]

	{ Key = Value } » Setting


parse(Line):
=========
» Line [text]
« Tree [list: “par”]

First parse line options at the end	-- separated like this
	Line.splitAt: “\t-- ”
		Before » Content [text]
		After » Header [text]

	Header.parse: » Tree.metadata.join:

Next, parse brackets. No nesting or overlapping.

Step 1: “Be fore [ Inner ] After” → { “Be”, “fore” }, “Inn er ] After”

	Content.splitAt: “[” or “_” or “**”
		Before » splitWords: { Label = “Word” } » Tree.join:
		Split » Bracket [text]
		After » Tail [text]

Step 2: { “Be”, “fore” }, “Inn er ] After” → { “Be”, “fore”, Variable = “Inn er” } + { After.parsed }

	Tail.splitAt: Bracket.inverse:
		Before » Inner [text]
		After » Remainder [text]
	
	Inner.parse: Bracket » Tree.join:
	Remainder.parse(Line): » Tree.join:


inverse(Bracket):
=============
» Bracket [text]
« Inverse [text]

	(Bracket == “[”) ⇒ “]”
	(Bracket == “_”) ⇒ “_”
	(Bracket == “**”) ⇒ “**”


parse(Inner):
==========
» Inner [text]
» Bracket [text]
« Result [list]

	(Bracket == “[”) ⇒ { Bracket.label: = Inner }
	(Bracket == “_”) ⇒ splitWords: { Block = Inner, Label = Bracket.label: }
	(Bracket == “**”) ⇒ splitWords: { Block = Inner, Label = Bracket.label: }


label(Bracket):
===========
» Bracket [text]
« Label [text]

	(Bracket == “[”) ⇒ “Variable”
	(Bracket == “_”) ⇒ “Italics”
	(Bracket == “**”) ⇒ “Bold”


splitWords:
=========
» Block [text]
» Label [text]
« Words [list]

	Block.splitAt: “ ”
		Before » Word [text]
		After » After [text]

	{ Label = Word } » Words.add:
	splitWords: { Block = After, Label } » Words.join:
