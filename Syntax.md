
parse (Template)
=====
	» Template [text]
	» Header [text]
	« Tree [list]

		Template.split: Line=“PANEL {…}”
			» map: (Panel → Panel.parse)
				» Temp [list]
		
		Temp.map: ({Tree [list], Remainder [text]} →
			Remainder.split: “\n” » map: (Line → Line.parse) » Tree.join)
				» Tree


parse (Panel)
=====
	» Panel [text]
	« Tree [list]
	« Remainder [text]

		parse: Header=Panel.metadata.“Splitter-after”.metadata.“Capture”
			» Tree.metadata.join
		
		Panel.splitOnce: Line={“{FOR} {…}”, “{IF} {…}”}
			Found » StartFound [yes/no]
			Before » OuterPre [text]
			Capture » {BlockLabel [text], BlockHeader [text]}
			After » InnerUnclosed [text]

		parse: Panel=InnerUnclosed
			Tree » InnerAParsed [list]
			Remainder » InnerBUnclosed [text]

		InnerBUnclosed.splitOnce: Line=BlockLabel.inverse
			Found » EndFound [yes/no]
			Before » InnerBUnparsed [text]
			After » OuterPost [text]


		either:
			o StartFound && EndFound ⇒ (
				OuterPre.split: “\n”
					» map: (Line → Line.parse)
						» Tree.join

				BracketLabel=(join:
					— InnerAParsed
					— InnerBUnparsed.split: “\n” » map: Line → Line.parse)
						» BracketContent [list]

				parse: Header=BlockHeader
					» BracketContent.metadata.join

				BracketContent » Tree.add
				OuterPost » Remainder)
			o otherwise ⇒
				Panel » Remainder


parse (Header)
=====
	» Header [text]
	« Options [list]

		Header.split: “, ”
			» map: {Option → Option.parse}
				» Options


parse (Option)
=====
	» Option [text]
	« Setting [text]

Create a key-value pair from “key=value” text.

		Option.splitOnce: ”=”
			Before » Key [text]
			After » Value [text]
	
		Key=Value » Setting


parse (Line)
=====
	» Line [text]
	« Tree [list “Par”]

First parse line options at the end	-- separated like this (after a tab-stop).

		Line.splitAt: “\t-- ”
			Before » Content [text]
			After » Header [text]
	
		Header.parse » Tree.metadata.join

Next, parse brackets. No nesting or overlapping.

		Content.splitAt: “[” or “_” or “**”
			Before » splitWords: Label=“Word” » Tree.join
			Split » Bracket [text]
			After » Tail [text]

		Tail.splitAt: Bracket.inverse
			Before » Inner [text]
			After » Remainder [text]
		
		parse: {Inner, Bracket} » Tree.join
		parse: Line=Remainder » Tree.join


label (Bracket)
=====
	» Bracket [text]
	« Label [text]

		either:
			o Bracket == “[”	⇒ “Variable”
			o Bracket == “_”	⇒ “Italics”
			o Bracket == “**”	⇒ “Bold”


inverse (Bracket)
=======
	» Bracket [text]
	« Inverse [text]

		either:
			o Bracket == “[”	⇒ “]”
			o Bracket == “_”	⇒ “_”
			o Bracket == “**”	⇒ “**”


parse (Inner)
=====
	» Inner [text]
	» Bracket [text]
	« Result [list]

Italics and bold text are split into words; variables aren’t.

		either:
			o Bracket == “[”	⇒ {Bracket.label=Inner}
			o Bracket == “_”	⇒ Inner.splitWords: Label=Bracket.label
			o Bracket == “**”	⇒ Inner.splitWords: Label=Bracket.label


splitWords [text]
==========
	» Block [text]
	» Label [text]
	« Words [list]

		Block.split: “ ”
			» map: (Word → Label=Word)
				» Words


---


mergeWith (Template)
=========
	» Template [list]
	» Data [list]
	« Doc [list]

		Template.map: (Panel → Panel.mergeWith: Data)
			» Doc

mergeWith (Panel)
=========
	» Panel [list]
	» Data [list]
	« Merged [list]

		Panel.map: (Par → Par.mergeWith: Data)
			» Merged

mergeWith (Par)
=========
	» Par [list]
	» Data [list]
	« Merged [list]

		either:
			o Par.key == “Par”	⇒
			o Par.key == “FOR”	⇒
			o Par.key == “IF”	⇒
