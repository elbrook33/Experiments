parse(Template)
=====
	» Template [text]
	» Header [text]

	« Tree [list]

		Template.split:
			— SplitterLine = “PANEL (…)¹”

		Splits » mapEach:
			— Panel [text] → (Panel.parse)

		Results » mapEach:
			— {Parsed [list], Tail [text]}
				→ Tail.parseLines » join: (Head = Parsed)

		Results » Tree

Compiles to the following C code:

```c
list parseTemplate(text Template, text Header)
{
	list Tree = {0};

	list Splits1 = split_SplitterLine(Template, "Panel (…)¹");
	list Results1 = mapEach(Splits1, parseTemplate_InlineFn1);
	list Results2 = mapEach(Results1, parseTemplate_InlineFn2);

	Tree = Results2;
	Tree = setKey(Tree, "Tree");

	delete_list(Splits1);
	delete_list(Results1);

	return Tree;
}
item parseTemplate_InlineFn1(item Panel)
{
	return parse_Panel(Panel);
}
item parseTemplate_InlineFn2(list Couple)
{
	list Parsed = itemAt(Couple, 1, "list");
	text Tail = itemAt(Couple, 2, "text");

	item Result = {0};

	list Parsed1 = parseLines(Tail);
	list Joined1 = join_list(Parsed, Parsed1);

	Result = Joined1;
	Result = setKey(Result, "Result");

	delete_list(Parsed1);

	return Result;
}
```


parse(Panel)
=====
	» Panel [text]
	« Tree [list]
	« Remainder [text]

		parseHeader:
			— Panel.metadata.“SplitterPre”.metadata.“Capture”

		Options » Tree.metadata.join


		Panel.splitOnce:
			— SplitterLines =
				— “(FOR)¹ (…)²”
				— “(IF)¹ (…)²”

		Found » StartFound [yes/no]
		Before » OuterPre [text]
		Capture » {BlockLabel [text], BlockHeader [text]}
		After » InnerUnclosed [text]


		InnerUnclosed.parsePanel

		Tree » InnerAParsed [list]
		Remainder » InnerBUnclosed [text]


		InnerBUnclosed.splitOnce:
			— SplitterLine = BlockLabel.inverse

		Found » EndFound [yes/no]
		Before » InnerBUnparsed [text]
		After » OuterPost [text]


		either:
			o StartFound && EndFound ⇒
				```
				OuterPre.parseLines » Tree.join

				InnerBUnparsed.parseLines » join: (Head = InnerAParsed)
				(BracketLabel = Joined) » BracketContent [list]
				BlockHeader.parseHeader » BracketContent.metadata.join

				BracketContent » Tree.add
				OuterPost » Remainder
				´´´

			o otherwise ⇒
				Panel » Remainder


parse(Header)
=====
	» Header [text]
	« Options [list]

		Header.split:
			— Splitter = “, ”

		Splits » mapEach:
			— Option [text] → Option.parse

		Result » Options


parse(Option)
=====
	» Option [text]
	« Setting [text]

Create a key-value pair from “key=value” text.

		Option.splitOnce:
			— Splitter = ”=”

		Before » Key [text]
		After » Value [text]

		(Key = Value) » Setting


parse(Lines)
=====
	» Lines [text]
	« Parsed [list]

		Line.split:
			— Splitter = “\n”

		Splits » mapEach:
			— Line [text] → Line.parse

		Results » Parsed


parse(Line)
=====
	» Line [text]
	« Tree [list “Par”]

First, parse line options at the end (separated by a tab and two hyphens).

		Line.splitOnce:
			— Splitter = “\t-- ”

		Before » Content [text]
		After » Header [text]

		Header.parse » Tree.metadata.join


Next, parse brackets. No nesting or overlapping.

		Content.splitOnce:
			— Splitters = {“[”, “_”, “**”}

		Before » splitWords: (Label = “Word”) » Tree.join
		Splitter » Bracket [text]
		After » Tail [text]


		Tail.splitOnce:
			— Splitter = Bracket.inverse

		Before » Inner [text]
		After » Remainder [text]
		

		Inner.parse: Bracket » Tree.join
		Remainder.parseLine » Tree.join


label(Bracket)
=====
	» Bracket [text]
	« Label [text]

		either:
			o Bracket == “[”	⇒ “Variable”
			o Bracket == “_”	⇒ “Italics”
			o Bracket == “**”	⇒ “Bold”


inverse(Bracket)
=======
	» Bracket [text]
	« Inverse [text]

		either:
			o Bracket == “[”	⇒ “]”
			o Bracket == “_”	⇒ “_”
			o Bracket == “**”	⇒ “**”


parse(Inner)
=====
	» Inner [text]
	» Bracket [text]
	« Result [list]

Italics and bold text are split into words; variables aren’t.

		either:
			o Bracket == “[”	⇒ {(Bracket.label = Inner)}
			o Bracket == “_”	⇒ Inner.splitWords: (Label = Bracket.label)
			o Bracket == “**”	⇒ Inner.splitWords: (Label = Bracket.label)


splitWords
==========
	» Block [text]
	» Label [text]
	« Words [list]

		Block.split:
			— Splitter = “ ”

		Splitter » mapEach:
			— Word [text] → (Label = Word)

		Results » Words


---


merge(Template)With
=========
	» Template [list]
	» Data [list]
	« Doc [list]

		Template.mapEach:
			— Panel [item] → (Panel.mergeWith: Data)

		Results » Doc


merge(Panel)With
=========
	» Panel [list]
	» Data [list]
	« Merged [list]

		Panel.mapEach:
			— Par [item] → (Par.mergeWith: Data)

		Results » Merged


merge(Par)With
=========
	» Par [list]
	» Data [list]
	« Merged [list]

		either:
			o Par.key == “Par”	⇒
			o Par.key == “FOR”	⇒
			o Par.key == “IF”	⇒
