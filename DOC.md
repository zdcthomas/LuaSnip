```
            __                       ____
           /\ \                     /\  _`\           __
           \ \ \      __  __     __ \ \,\L\_\    ___ /\_\  _____
            \ \ \  __/\ \/\ \  /'__`\\/_\__ \  /' _ `\/\ \/\ '__`\
             \ \ \L\ \ \ \_\ \/\ \L\.\_/\ \L\ \/\ \/\ \ \ \ \ \L\ \
              \ \____/\ \____/\ \__/.\_\ `\____\ \_\ \_\ \_\ \ ,__/
               \/___/  \/___/  \/__/\/_/\/_____/\/_/\/_/\/_/\ \ \/
                                                             \ \_\
                                                              \/_/
```

Luasnip is a snippet-engine written entirely in lua. It has some great
features like inserting text (`luasnip-function-node`) or nodes
(`luasnip-dynamic-node`) based on user input, parsing LSP syntax and switching
nodes (`luasnip-choice-node`).
For basic setup like mappings and installing, check the README.

All code-snippets in this help assume that

```lua
local ls = require"luasnip"
local s = ls.snippet
local sn = ls.snippet_node
local isn = ls.indent_snippet_node
local t = ls.text_node
local i = ls.insert_node
local f = ls.function_node
local c = ls.choice_node
local d = ls.dynamic_node
local r = ls.restore_node
local events = require("luasnip.util.events")
local ai = require("luasnip.nodes.absolute_indexer")
local fmt = require("luasnip.extras.fmt").fmt
local m = require("luasnip.extras").m
local lambda = require("luasnip.extras").l
local postfix = require("luasnip.extras.postfix").postfix
```

# BASICS
In LuaSnip, snippets are made up of `nodes`. These can contain either

- static text (`textNode`)
- text that can be edited (`insertNode`)
- text that can be generated from the contents of other nodes (`functionNode`)
- other nodes
    - `choiceNode`: allows choosing between two nodes (which might contain more
    nodes)
    - `restoreNode`: store and restore input to nodes
- or nodes that can be generated based on input (`dynamicNode`).

Snippets are always created using the `s(trigger:string, nodes:table)`-function.
It is explained in more detail in [SNIPPETS](#snippets), but the gist is that
it creates a snippet that contains the nodes specified in `nodes`, which will be
inserted into a buffer if the text before the cursor matches `trigger` when
`expand` is called.
The snippets for a given filetype have to be added to luasnip via
`ls.add_snippets(filetype, snippets)`. Snippets that should be accessible
globally (in all filetypes) have to be added to the special filetype `all`.
```lua
ls.add_snippets("all", {
	s("ternary", {
		-- equivalent to "${1:cond} ? ${2:then} : ${3:else}"
		i(1, "cond"), t(" ? "), i(2, "then"), t(" : "), i(3, "else")
	})
})
```
It is possible to make snippets from one filetype available to another using
`ls.filetype_extend`, more info on that [here](#api-reference).

# NODE

Every node accepts, as its' last parameter, an optional table of arguments.
There are some common ones (e.g. [`node_ext_opts`](#ext_opts)), and some that
only apply to some nodes (`user_args` for both function and dynamicNode).
These `opts` are only mentioned if they accept options that are not common to
all nodes.

# SNIPPETS

The most direct way to define snippets is `s`:
```lua
s({trig="trigger"}, {})
```

(This snippet is useless beyond being a minimal example)

`s` accepts, as the first argument, a table with the following possible
entries:

- `trig`: string, plain text by default. The only entry that must be given.
- `name`: string, can be used by e.g. `nvim-compe` to identify the snippet.
- `dscr`: string, description of the snippet, \n-separated or table
  for multiple lines.
- `wordTrig`: boolean, if true, the snippet is only expanded if the word
  (`[%w_]+`) before the cursor matches the trigger entirely.
  True by default.
- `regTrig`: boolean, whether the trigger should be interpreted as a
  lua pattern. False by default.
- `docstring`: string, textual representation of the snippet, specified like
  `dscr`. Overrides docstrings loaded from json.
- `docTrig`: string, for snippets triggered using a lua pattern: define the
  trigger that is used during docstring-generation.
- `hidden`: hint for completion-engines, if set, the snippet should not show
  up when querying snippets.
- `priority`: Priority of the snippet, a positive number, 1000 by default.
  Snippets with high priority will be matched to a trigger before those with a
  lower one.
  The priority for multiple snippets can also be set in `add_snippets`.

`s` can also be a single string, in which case it is used instead of `trig`, all
other values being defaulted:

```lua
s("trigger", {})
```

The second argument to `s` is a table containing all nodes that belong to the
snippet. If the table only has a single node, it can be passed directly
without wrapping it in a table.

The third argument (`opts`) is a table with the following valid keys:

- `condition`: the condition-function `fn(line_to_cursor, matched_trigger,
  captures) -> bool`.
  The snippet will be expanded only if it returns true (default is a function
  that just returns `true`).
  The function is called before the text is modified in any way.
  Some parameters are passed to the function: The line up to the cursor, the
  matched trigger, and the captures (table).
- `show_condition`: Function with signature `f(line_to_cursor) -> bool`.
  It is a hint for completion-engines, indicating when the snippet should be
  included in current completion candidates.
  Defaults to a function returning `true`.
  This is different from `condition` because `condition` is evaluated by
  LuaSnip on snippet expansion (and thus has access to the matched trigger and
  captures), while `show_condition` is evaluated by the completion-engine when
  scanning for available snippet candidates.
- `callbacks`: Contains functions that are called upon enterin/leaving a node
  of this snippet.
  To print text upon entering the _second_ node of a snippet, `callbacks`
  should be set as follows:
  ```lua
  {
  	-- position of the node, not the jump-position!!
  	-- s("trig", {t"first node", t"second node", i(1, "third node")}).
  	[2] = {
  		[events.enter] = function(node, _event_args) print("2!") end
  	}
  }
  ```
  To register a callback for the snippets' own events, the key `[-1]` may
  be used.
  More info on events [here](#here)
- `child_ext_opts`, `merge_child_ext_opts`: `ext_opts` applied to the children
  of this snippet. More info [here](#ext_opts).

This `opts`-table can also be passed to e.g.	`snippetNode` or `indentSnippetNode`,
but only `callbacks` and the `ext_opts`-related options are used there.

Snippets contain some interesting tables, e.g. `snippet.env` contains variables
used in the LSP-protocol like `TM_CURRENT_LINE` or `TM_FILENAME` or
`snippet.captures`, where capture-groups of regex-triggers are stored.
Additionally, the string that was used to trigger the snippet is stored in
`snippet.trigger`. These variables/tables are primarily useful in
dynamic/functionNodes, where the snippet can be accessed through the immediate
parent (`parent.snippet`), which is passed to the function.

## Api:

- `invalidate()`: call this method to effectively remove the snippet. The
  snippet will no longer be able to expand via `expand` or `expand_auto`. It
  will also be hidden from lists (at least if the plugin creating the list
  respects the `hidden`-key), but it might be necessary to call
  `ls.refresh_notify(ft)` after invalidating snippets.


# TEXTNODE

The most simple kind of node; just text.
```lua
s("trigger", { t("Wow! Text!") })
```
This snippet expands to

```
    Wow! Text!⎵
```
Where ⎵ is the cursor.
Multiline-strings can be defined by passing a table of lines rather than a
string:

```lua
s("trigger", {
	t({"Wow! Text!", "And another line."})
})
```


# INSERTNODE

These Nodes contain editable text and can be jumped to- and from (e.g.
traditional placeholders, like `$1` in textmate-snippets).

The functionality is best demonstrated with an example:

```lua
s("trigger", {
	t({"After expanding, the cursor is here ->"}), i(1),
	t({"", "After jumping forward once, cursor is here ->"}), i(2),
	t({"", "After jumping once more, the snippet is exited there ->"}), i(0),
})
```

The InsertNodes are jumped over in order from `1 to n`.
The 0-th node is special as it's always the last one.
So the order of InsertNode jump is as follows:

1. After expansion, we will be at InsertNode 1.
2. After jumping forward, we will be at InsertNode 2.
3. After jumping forward again, we will be at InsertNode 0.

If no 0-th InsertNode is found in a snippet, one is automatically inserted
after all other nodes.

The jumping-order doesn't have to follow the "textual" order of the nodes:
```lua
s("trigger", {
	t({"After jumping forward once, cursor is here ->"}), i(2),
	t({"", "After expanding, the cursor is here ->"}), i(1),
	t({"", "After jumping once more, the snippet is exited there ->"}), i(0),
})
```
The above snippet will behave as follows:

1. After expansion, we will be at InsertNode 1.
2. After jumping forward, we will be at InsertNode 2.
3. After jumping forward again, we will be at InsertNode 0.

An **important** (because here luasnip differs from other snippet-engines) detail
is that the jump-positions restart at 1 in nested snippets:
```lua
s("trigger", {
	i(1, "First jump"),
	t(" :: "),
	sn(2, {
		i(1, "Second jump"),
		t" : ",
		i(2, "Third jump")
	})
})
```

as opposed to e.g. the textmate-syntax, where tabstops are snippet-global:
```snippet
${1:First jump} :: ${2: ${3:Third jump} : ${4:Fourth jump}}
```
(this is not exactly the same snippet of course, but as close as possible)
(the restart-rule only applies when defining snippets in lua, the above
textmate-snippet will expand correctly).

It's possible to have initial text inside an InsertNode, which is comfortable
for potentially keeping some default-value:
```lua
	s("trigger", i(1, "This text is SELECTed after expanding the snippet."))
```
This initial text is defined the same way as textNodes, e.g. can be multiline.

`i(0)`s can have initial text, but do note that when the SELECTed text is
replaced, its' replacement won't end up in the `i(0)`, but behind it (for
reasons, check out Luasnip#110).


# FUNCTIONNODE

Function Nodes insert text based on the content of other nodes using a
user-defined function:
```lua
 s("trig", {
 	i(1),
 	f(function(args, snip, user_arg_1) return args[1][1] .. user_arg_1 end,
 		{1},
 		{ user_args = {"Will be appended to text from i(0)"}}),
 	i(0)
 })
```
The first parameter of `f` is the function. Its parameters are:

1. A table of the text of currently contained in the argnodes.
      (e.g. `{{line1}, {line1, line2}}`). The snippet-indent will be removed from
      all lines following the first.

2. The immediate parent of the `functionNode`. It is included here as it allows
      easy access to anything that could be useful in functionNodes (i.e.
      `parent.snippet.env` or `parent.snippet.captures`, which contains capture
      groups of regex-triggered snippets). In most cases `parent.env` works,
      but if a `functionNode` is nested within a `snippetNode`, the immediate
      parent (a `snippetNode`) will contain neither `captures` nor `env`. Those
      are only stored in the `snippet`, which can be accessed as `parent.snippet`.

3. The `user_args` passed in `opts`. Note that there may be multiple user_args (e.g. `user_args1, ..., user_argsn`).

The function shall return a string, which will be inserted as-is, or a table
of strings for multiline-string, here all lines following the first will be
prefixed with the snippets' indentation.


The second parameter is a table of indices of jumpable nodes whose text is
passed to the function.
The table may be empty, in this case the function is evaluated once upon
snippet-expansion.
If the table only has a single node, it can be passed directly without wrapping
it in a table.
The indices can be specified either as relative to the functionNodes' parent
using numbers or as absolute, using the [`absolute_indexer`](#absolute_indexer).

The last parameter is, as with any node, `opts`.
`functionNode` accepts one additional option: `user_args`, a table of values
passed to the function.
These exist to more easily reuse functionNode-functions, when applicable:

```lua
local function reused_func(_,_, user_arg1)
	return user_arg1
end

s("trig", {
	f(reused_func, {}, {
		user_args = {"text"}
	}),
	f(reused_func, {}, {
		user_args = {"different text"}
	}),
})
```

Examples:
Use captures from the regex-trigger using a functionNode:

```lua
s({trig = "b(%d)", regTrig = true},
	f(function(args, snip) return
		"Captured Text: " .. snip.captures[1] .. "." end, {})
)
```

The table passed to functionNode:

```lua
s("trig", {
	i(1, "text_of_first"),
	i(2, {"first_line_of_second", "second_line_of_second"}),
	-- order is 2,1, not 1,2!!
	f(function(args, snip)
		--here
	end, {2, 1} )})
```

At `--here`, `args` would look as follows (provided no text was changed after
expansion):
```lua
args = {
	{"first_line_of_second", "second_line_of_second"},
	{"text_of_first"}
}
```

One more example to show usage of `absolute_indexer`:
```lua
s("trig", {
	i(1, "text_of_first"),
	i(2, {"first_line_of_second", "second_line_of_second"}),
	f(function(args, snip)
		-- just concat first lines of both.
		return args[1][1] .. args[2][1]
	end, {ai[2], ai[1]} )})
```

If the function only performs simple operations on text, consider using
the `lambda` from [`luasnip.extras`](#extras)

# POSTFIX SNIPPET

Postfix snippets, famously used in 
[rust analyzer](https://rust-analyzer.github.io/) and various IDEs, are a type
of snippet, which alters text before the snippets trigger. While these
can be implemented using regTrig snippets, this helper makes the process easier
in most cases.

The simplest example, which surrounds the text preceeding the `.br` with
brackets `[]`, looks like:

```lua

      postfix(".br", {
              f(function(_, parent)
                    return "[" .. parent.snippet.env.POSTFIX_MATCH .. "]"
              end, {}),
      })
```

and is triggered with `xxx.br` and expands to `[xxx]`.

Note the `parent.snippet.env.POSTFIX_MATCH` in the function node. This is additional
field generated by the postfix snippet. This field is generated by extracting
the text matched (using a configurable matching string, see below) from before
the trigger. In the case above, the field would equal `"xxx"`. This is also
usable within dynamic nodes.

This field can also be used within lambdas and dynamic nodes.

```lua
      postfix(".br", {
              l("[" .. l.POSTFIX_MATCH .. "]"),
      })
```

```lua
	postfix(".brd", {
	  d(1, function (_, parent)
	    return sn(nil, {t("[" .. parent.env.POSTFIX_MATCH .. "]")})
	  end)
	}),
```

The arguments to `postfix` are identical to the arguments to `s` but with a few
extra options.

The first argument can be either a string or a table. If it is a string, that
string will act as the trigger, and if it is a table it has the same valid keys
as the table in the same position for `s` except:

- `wordTrig`: This key will be ignored if passed in, as it must always be
    false for postfix snippets.
- `match_pattern`: The pattern that the line before the trigger is matched
    against. The default match pattern is `"[%w%.%_%-]+$"`. Note the `$`. This
    matches since only the line _up until_ the beginning of the trigger is
    matched against the pattern, which makes the character immediately
    preceeding the trigger match as the end of the string.

Some other match strings, including the default, are available from the postfix
module. `require("luasnip.extras.postfix).matches`:

- `default`: [%w%.%_%-%"%']+$
- 'line': `^.+$`

The second argument is identical to the second argument for `s`, that is, a
table of nodes.

The optional third argument is the same as the third (`opts`) argument to the
`s` function, but with one difference:

The postfix snippet works using a callback on the pre_expand event of the
snippet. If you pass a callback on the pre_expand event (structure example
below) it will get run after the builtin callback. This means that your
callback will have access to the `POSTFIX_MATCH` field as well.

```lua

      {
        callbacks = {
              [-1] = {
                      [events.pre_expand] = function(snippet, event_args)
                          -- function body to match before the dot
                          -- goes here
                       end
                      }
                    }
        }
```

# CHOICENODE

ChoiceNodes allow choosing between multiple nodes.

```lua
 s("trig", c(1, {
 	t("Ugh boring, a text node"),
 	i(nil, "At least I can edit something now..."),
 	f(function(args) return "Still only counts as text!!" end, {})
 }))
```

`c()` expects as its first arg, as with any jumpable node, its position in the
jumplist, and as its second a table with nodes, the choices. This table can
either contain a single node or a table of nodes. In the latter case the table
will be converted into a `snippetNode`.
The third parameter is a table of options with the following keys:

- `restore_cursor`: `false` by default. If it is set, and the node that was
	being edited also appears in the switched-to choice (can be the case if a
	`restoreNode` is present in both choice) the cursor is restored relative to
	that node.
	The default is `false` as enabling might lead to worse performance. It's
	possible to override the default by wrapping the `choiceNode`-constructor
	in another function that sets `opts.restore_cursor` to `true` and then using
	that to construct `choiceNode`s:
    ```lua
    local function restore_cursor_choice(pos, choices, opts)
        if opts then
            opts.restore_cursor = true
        else
            opts = {restore_cursor = true}
        end
        return c(pos, choices, opts)
    end
    ```

Jumpable nodes that normally expect an index as their first parameter don't
need one inside a choiceNode; their index is the same as the choiceNodes'.

As it is only possible (for now) to change choices from within the choiceNode,
make sure that all of the choices have some place for the cursor to stop at.
This means that in `sn(nil, {...nodes...})` `nodes` has to contain e.g. an
`i(1)`, otherwise luasnip will just "jump through" the nodes, making it
impossible to change the choice.

```lua
c(1, {
	t"some text", -- textNodes are just stopped at.
	i(nil, "some text"), -- likewise.
	sn(nil, {t"some text"}) -- this will not work!
	sn(nil, {i(1), t"some text"}) -- this will.
})
```

The active choice for a choiceNode can be changed by calling `ls.change_choice(1)`
(forwards) or `ls.change_choice(-1)` (backwards), for example via

```lua
-- set keybinds for both INSERT and VISUAL.
vim.api.nvim_set_keymap("i", "<C-n>", "<Plug>luasnip-next-choice", {})
vim.api.nvim_set_keymap("s", "<C-n>", "<Plug>luasnip-next-choice", {})
vim.api.nvim_set_keymap("i", "<C-p>", "<Plug>luasnip-prev-choice", {})
vim.api.nvim_set_keymap("s", "<C-p>", "<Plug>luasnip-prev-choice", {})
```

Apart from this, there is also a [picker](#select_choice) where no cycling is necessary and any
choice can be selected right away, via `vim.ui.select`.

# SNIPPETNODE

SnippetNodes directly insert their contents into the surrounding snippet.
This is useful for choiceNodes, which only accept one child, or dynamicNodes,
where nodes are created at runtime and inserted as a snippetNode.

Syntax is similar to snippets, however, where snippets require a table
specifying when to expand, snippetNodes, similar to insertNodes, expect a
number, as they too are jumpable:
```lua
 s("trig", sn(1, {
 	t("basically just text "),
 	i(1, "And an insertNode.")
 }))
```

Note that snippetNodes don't expect an `i(0)`.



# INDENTSNIPPETNODE

By default, all nodes are indented at least as deep as the trigger. With these
nodes it's possible to override that behaviour:

```lua
s("isn", {
	isn(1, {
		t({"This is indented as deep as the trigger",
		"and this is at the beginning of the next line"})
	}, "")
})
```

(Note the empty string passed to isn).

Indent is only applied after linebreaks, so it's not possible to remove indent
on the line where the snippet was triggered using `ISN` (That is possible via
regex-triggers where the entire line before the trigger is matched).

Another nice usecase for `ISN` is inserting text, e.g. `//` or some other comment-
string before the nodes of the snippet:

```lua
s("isn2", {
	isn(1, t({"//This is", "A multiline", "comment"}), "$PARENT_INDENT//")
})
```

Here the `//` before `This is` is important, once again, because indent is only
applied after linebreaks.
To enable such usage, `$PARENT_INDENT` in the indentstring is replaced by the
parents' indent (duh).



# DYNAMICNODE

Very similar to functionNode, but returns a snippetNode instead of just text,
which makes them very powerful as parts of the snippet can be changed based on
user-input.

The prototype for the dynamicNodes' constructor is 
`d(position:int, function, argnodes:table of nodes, opts: table)`:

1. `position`: just like all jumpable nodes, when this node will be jumped into.
2. `function`: `fn(args, parent, old_state, user_args1, ..., user_argsn) -> snippetNode`
   This function is called when the argnodes' text changes. It generates and
   returns (wrapped inside a `snippetNode`) the nodes that should be inserted
   at the dynamicNodes place.
   `args`, `parent` and `user_args` are also explained in
   [functionNode](#functionnode)
   * `args`: `table of text` (`{{"node1line1", "node1line2"}, {"node2line1"}}`)
     from nodes the dynamicNode depends on.
   * `parent`: the immediate parent of the `dynamicNode`).
   * `old_state`: a user-defined table. This table may contain
     anything, its intended usage is to preserve information from the previously
     generated `snippetNode`: If the `dynamicNode` depends on other nodes it may
     be reconstructed, which means all user input (text inserted in `insertNodes`,
     changed choices) to the previous dynamicNode is lost.
     The `old_state` table must be stored in `snippetNode` returned by
     the function (`snippetNode.old_state`).
     The second example below illustrates the usage of `old_state`.
   * `user_args1, ..., user_argsn`: passed through from `dynamicNode`-opts.
3. `argnodes`: Indices of nodes the dynamicNode depends on: if any of these trigger an
   update, the `dynamicNode`s' function will be executed, and the result inserted at
   the `dynamicNodes` place.
   Can be a single index or a table of indices.
4. `opts`: Just like `functionNode`, `dynamicNode` also accepts `user_args` in
   addition to options common to all nodes.

Examples:

```lua
s("trig", {
	t"text: ", i(1), t{"", "copy: "},
	d(2, function(args)
			-- the returned snippetNode doesn't need a position; it's inserted
			-- "inside" the dynamicNode.
			return sn(nil, {
				-- jump-indices are local to each snippetNode, so restart at 1.
				i(1, args[1])
			})
		end,
	{1})
})
```

This `dynamicNode` inserts an `insertNode` which copies the text inside the
first `insertNode`.

```lua
local function lines(args, parent, old_state, initial_text)
	local nodes = {}
	old_state = old_state or {}

	-- count is nil for invalid input.
	local count = tonumber(args[1][1])
	-- Make sure there's a number in args[1].
	if count then
		for j=1, count do
			local iNode
			if old_state and old_state[j] then
				-- old_text is used internally to determine whether
				-- dependents should be updated. It is updated whenever the
				-- node is left, but remains valid when the node is no
				-- longer 'rendered', whereas node:get_text() grabs the text
				-- directly from the node.
				iNode = i(j, old_state[j].old_text)
			else
			  iNode = i(j, initial_text)
			end
			nodes[2*j-1] = iNode

			-- linebreak
			nodes[2*j] = t({"",""})
			-- Store insertNode in old_state, potentially overwriting older
			-- nodes.
			old_state[j] = iNode
		end
	else
		nodes[1] = t("Enter a number!")
	end

	local snip = sn(nil, nodes)
	snip.old_state = old_state
	return snip
end

...

s("trig", {
	i(1, "1"),
	-- pos, function, argnodes, opts (containing the user_arg).
	d(2, lines, {1}, {user_args = {"Sample Text"}})
})
```
This snippet would start out as `"1\nSample Text"` and, upon changing the 1 to
e.g. 3, it would change to `"3\nSample Text\nSample Text\nSample Text"`. Text
that was inserted into any of the dynamicNodes insertNodes is kept when
changing to a bigger number.
(`old_state` is no longer the best way to preserve user-input across multiple
recreations: the shortly-explained `restoreNode` is much more user-friendly)

# RESTORENODE

This node can store and restore a snippetNode that was modified (changed
choices, inserted text) by the user. Its usage is best demonstrated by an
example:

```lua
s("paren_change", {
	c(1, {
		sn(nil, { t("("), r(1, "user_text"), t(")") }),
		sn(nil, { t("["), r(1, "user_text"), t("]") }),
		sn(nil, { t("{"), r(1, "user_text"), t("}") }),
	}),
}, {
	stored = {
		user_text = i(1, "default_text")
	}
})
```

Here the text entered into `user_text` is preserved upon changing choice.

The constructor for the restoreNode, `r`, takes (at most) three parameters:
- `pos`, when to jump to this node.
- `key`, the key that identifies which `restoreNode`s should share their
  content.
- `nodes`, the contents of the `restoreNode`. Can either be a single node, or
  a table of nodes (both of which will be wrapped inside a `snippetNode`,
  except if the single node already is a `snippetNode`).
  The content of a given key may be defined multiple times, but if the
  contents differ, it's undefined which will actually be used.
  If a keys content is defined in a `dynamicNode`, it will not be used for
  `restoreNodes` outside that `dynamicNode`. A way around this limitation is
  defining the content in the `restoreNode` outside the `dynamicNode`.

The content for a key may also be defined in the `opts`-parameter of the
snippet-constructor, as seen in the example above. The `stored`-table accepts
the same values as the `nodes`-parameter passed to `r`.
If no content is defined for a key, it defaults to the empty `insertNode`.

An important-to-know limitation of `restoreNode` is that, for a given key, only
one may be visible at a time. See
[this issue](https://github.com/L3MON4D3/LuaSnip/issues/234) for details.

The `restoreNode` is also useful for storing user-input across updates of a
`dynamicNode`. Consider this:

```lua
local function simple_restore(args, _)
	return sn(nil, {i(1, args[1]), i(2, "user_text")})
end

s("rest", {
	i(1, "preset"), t{"",""},
	d(2, simple_restore, 1)
}),
```

Every time the `i(1)` in the outer snippet is changed, the text inside the
`dynamicNode` is reset to `"user_text"`. This can be prevented by using a
`restoreNode`:

```lua
local function simple_restore(args, _)
	return sn(nil, {i(1, args[1]), r(2, "dyn", i(nil, "user_text"))})
end

s("rest", {
	i(1, "preset"), t{"",""},
	d(2, simple_restore, 1)
}),
```
Now the entered text is stored.

`RestoreNode`s indent is not influenced by `indentSnippetNodes` right now. If
that really bothers you feel free to open an issue.

# ABSOLUTE_INDEXER

The `absolute_indexer` can be used to pass text of nodes to a function/dynamicNode
that it doesn't share a parent with.
Normally, accessing the outer `i(1)` isn't possible from inside e.g. a
snippetNode (nested inside a choiceNode to make this example more practical):

```lua
s("trig", {
	i(1), c(2, {
		sn(nil, {
			t"cannot access the argnode :(", f(function(args) return args[1] end, {???})
		}),
		t"sample_text"
	})
})
```

Using `absolute_indexer`, it's possible to do so:
```lua
s("trig", {
	i(1), c(2, {
		sn(nil, { i(1),
			t"can access the argnode :)", f(function(args) return args[1] end, ai[1])
		}),
		t"sample_text"
	})
})
```

There are some quirks in addressing nodes:
```lua
s("trig", {
	i(2), -- ai[2]: indices based on insert-order, not position.
	sn(1, { -- ai[1]
		i(1), -- ai[1][1]
		t"lel", -- not addressable.
		i(2) -- ai[1][2]
	}),
	c(3, { -- ai[3]
		i(nil), -- ai[3][1]
		t"lel", -- ai[3][2]: choices are always addressable.
	}),
	d(4, function() -- ai[4]
		return sn(nil, { -- ai[4][0]
			i(1), -- ai[4][0][1]
		})
	end, {})
	}))
	r(5, "restore_key", -- ai[5]
		i(1) -- ai[5][0][1]: restoreNodes always store snippetNodes.
	)
	r(6, "restore_key_2", -- ai[6]
		sn(nil, { -- ai[6][0]
			i(1) -- ai[6][0][1]
		})
	)
	}))
})
```

Note specifically that the index of a dynamicNode differs from that of the
generated snippetNode, and that restoreNodes (internally) always store a
snippetNode, so even if the restoreNode only contains one node, that node has
to be accessed as `ai[restoreNodeIndx][0][1]`.

`absolute_indexer`s' can be constructed in different ways:
```lua
ai[1][2][3] == ai(1, 2, 3) == ai{1, 2, 3}
```

# EXTRAS

The module `"luasnip.extras"` contains nodes that ease writing snippets (This
is only a short outline, their usage is shown more expansively in
`Examples/snippets.lua`):

- `lambda`: A shortcut for `functionNode`s that only do very basic string-
manipulation. For example, to replace all occurences of "a" in the nth insert
with "e", one could use `lambda(lambda._1:gsub("a", "e"), n)` (signature is
similar to that of `functionNode`).
If a node has multiple lines, they will be concatenated using "\n".

- `match`: Can insert text based on a predicate (shorthand for `functionNode`s).
The complete signature for the node is `match(argnodes, condition, then, else)`, where
  * `argnodes` can be specified as in `functionNode`,
  * `condition` may be a
    * string: interpreted as a lua-pattern. Matched on the `\n`-joined (in case
      it's multiline) text of the first argnode (`args[1]:match(condition)`).
    * function: `fn(args, snip) -> bool`: takes the same parameters as the
      `functionNode`-function, any value other than nil or false is interpreted
      as a match.
    * lambda: `l._n` is the `\n`-joined text of the nth argnode.
      Useful if string-manipulations have to be performed before the string is matched.

  * `then` is inserted if the condition matches, `else` if it doesn't. They can
  	both be either text, lambda or function (with the same parameters as
  	specified above).
  If `then` is not given, the `then`-value depends on what was specified as the
  `condition`:
    * pattern: Simply the return value from the `match`, e.g. the entire match,
    or, if there were capture groups, the first capture group.
    * function: the return value of the function if it is either a string, or a
    table (if there is no `then`, the function cannot return a table containing
    something other than strings).
    * lambda: Simply the first value returned by the lambda.

  Examples:
  * `match(n, "^ABC$", "A")` inserts "A" if the `n`th jumpable node matches
    "ABC" exactly, nothing otherwise.
  * `match(n, lambda._1:match(lambda._1:reverse()), "PALINDROME")` inserts
    "PALINDROME" if the nth jumpable node is a palindrome.

  * ```lua
    s("trig", {
    	i(1), t":",
    	i(2), t"::",
    	m({1, 2}, lambda._1:match("^"..lambda._2.."$"), lambda._1:gsub("a", "e"))
    })
    ```
    This inserts the text of the first insertNode, with all occurences of `a`
    replaced with `e` if the second insertNode matches the first exactly.

- `rep`: repeats the node with the passed index. `rep(1)` to repeat the content
of the first insert.

- `partial`: directly inserts the output of a function. Useful for e.g.
`partial(os.date, "%Y")` (arguments passed after the function are passed to it).

- `nonempty`: inserts text if the insert at the given index doesn't contain any
text. `nonempty(n, "empty!", "not empty!")` inserts "empty!" if insert n is
empty, "not empty!" if it isn't.

- `dynamic_lambda`: Operates almost exactly like `lambda`, only that it can be
jumped to, and it's contents therefore be easily overridden.
`dynamic_lambda(2, lambda._1..lambda._1, 1)` will first contain the content of
insert 1 appended to itself, but the second jump will lead to it, making it
easy to override the generated text.
The text will only be changed when a argnode updates it.

## FMT

`require("luasnip.extras.fmt").fmt` can be used to create snippets in a more
readable way.

Simple example:

```lua
ls.add_snippets("all", {
	-- important! fmt does not return a snippet, it returns a table of nodes.
	s("example1", fmt("just an {iNode1}", {
		iNode1 = i(1, "example")
	}),
	s("example2", fmt([[
		if {} then
			{}
		end
	]], {
		-- i(1) is at nodes[1], i(2) at nodes[2].
		i(1, "not now"), i(2, "when")
	}),
	s("example3", fmt([[
		if <> then
			<>
		end
	]], {
		-- i(1) is at nodes[1], i(2) at nodes[2].
		i(1, "not now"), i(2, "when")
	}, {
		delimiters = "<>"
	}),
})
```

`fmt(format:string, nodes:table of nodes, opts:table|nil) -> table of nodes`

* `format`: a string. Occurences of `{<somekey>}` ( `{,}` are customizable, more
  on that later) are replaced with `content[<somekey>]` (which should be a
  node), while surrounding text becomes `textNode`s.  
  To escape a delimiter, repeat it (`"{{"`).  
  If no key is given (`{}`) are numbered automatically:  
  `"{} ? {} : {}"` becomes `"{1} ? {2} : {3}"`, while
  `"{} ? {3} : {}"` becomes `"{1} ? {3} : {4}"` (the count restarts at each
  numbered placeholder).
  If a key appears more than once in `format`, the node in
  `content[<duplicate_key>]` is inserted for the first, and copies of it for
  subsequent occurences.
* `nodes`: just a table of nodes.
* `opts`: optional arguments:
  * `delimiters`: string, two characters. Change `{,}` to some other pair, e.g.
  	`"<>"`.
  * `strict`: Warn about unused nodes (default true).
  * `trim_empty`: remove empty (`"%s*"`) first and last line in `format`. Useful
  	when passing multiline strings via `[[]]` (default true).
  * `dedent`: remove indent common to all lines in `format`. Again, makes
  	passing multiline-strings a bit nicer (default true).

There is also `require("luasnip.extras.fmt").fmta`. This only differs from `fmt`
by using angle-brackets (`<>`) as the default-delimiter.


## On The Fly snippets
You can create snippets that are not for being used all the time but only
in a single session.

This behaves as an "operator" takes what is in a register and transforms it into a
snippet using words prefixed as $ as inputs or copies (depending on if the same word appears 
more than once). You can escape $ by repeating it.

In order to use, add something like this to your config:
```vim
vnoremap <c-f>  "ec<cmd>lua require('luasnip.extras.otf').on_the_fly()<cr>
inoremap <c-f>  <cmd>lua require('luasnip.extras.otf').on_the_fly("e")<cr>
```

Notice that you can use your own mapping instead of <c-f>, and you can pick another register
instead of `"p`. You can even use it several times, as if it where a macro if you add several
mapppings like:
```vim
; For register a
vnoremap <c-f>a  "ac<cmd>lua require('luasnip.extras.otf').on_the_fly()<cr
inoremap <c-f>a  <cmd>lua require('luasnip.extras.otf').on_the_fly("a")<cr>


; For register b
vnoremap <c-f>a  "bc<cmd>:lua require('luasnip.extras.otf').on_the_fly()<cr
inoremap <c-f>b  <cmd>lua require('luasnip.extras.otf').on_the_fly("b")<cr>
```

## Select_choice

It's possible to leverage `vim.ui.select` for selecting a choice directly,
without cycling through choices.
All that is needed for this is calling `require("luasnip.extras.select_choice")`,
preferably via some keybind, e.g.

```vim
inoremap <c-u> <cmd>lua require("luasnip.extras.select_choice")()<cr>
```
, while inside a choiceNode. The `opts.kind` hint for `vim.ui.select` will be set to `luasnip`.


## filetype_functions

Contains some utility-functions that can be passed to the `ft_func` or
`load_ft_func`-settings.

* `from_filetype`: the default for `ft_func`. Simply returns the filetype(s) of
  the buffer.
* `from_cursor_pos`: uses treesitter to determine the filetype at the cursor.
  With that, it's possible to expand snippets in injected regions, as long as
  the treesitter-parser supports them.
  If this is used in conjuction with `lazy_load`, extra care must be taken that
  all the filetypes that can be expanded in a given buffer are also returned by
  `load_ft_func` (otherwise their snippets may not be loaded).
  This can easily be achieved with `extend_load_ft`.
* `extend_load_ft`: `fn(extend_ft:map) -> fn`
  A simple solution to the problem described above is loading more filetypes
  than just that of the target-buffer when `lazy_load`ing. This can be done
  ergonomically via `extend_load_ft`: calling it with a table where the keys are
  filetypes, and the values are the filetypes that should be loaded additionaly
  returns a function that can be passed to `load_ft_func` and takes care of
  extending the filetypes properly.

  ```lua
  ls.config.setup({
  	load_ft_func =
  		-- Also load both lua and json when a markdown-file is opened,
  		-- javascript for html.
  		-- Other filetypes just load themselves.
  		require("luasnip.extras.filetype_functions").extend_load_ft({
  			markdown = {"lua", "json"},
  			html = {"javascript"}
  		})
  })
  ```

# LSP-SNIPPETS

Luasnip is capable of parsing lsp-style snippets using
`ls.parser.parse_snippet(context, snippet_string)`:
```lua
ls.parser.parse_snippet({trig = "lsp"}, "$1 is ${2|hard,easy,challenging|}")
```

Nested placeholders(`"${1:this is ${2:nested}}"`) will be turned into
choiceNode's with:
	- the given snippet(`"this is ${1:nested}"`) and
	- an empty insertNode



# VARIABLES

All `TM_something`-variables are supported with two additions:
`SELECT_RAW` and `SELECT_DEDENT`. These were introduced because
`TM_SELECTED_TEXT` is designed to be compatible with vscodes' behavior, which
can be counterintuitive when the snippet can be expanded at places other than
the point where selection started (or when doing transformations on selected text).

All variables can be used outside of lsp-parsed snippets as their values are
stored in a snippets' `snip.env`-table:
```lua
s("selected_text", {
	-- the surrounding snippet is passed in args after all argnodes (none,
	-- in this case).
	f(function(args, snip) return snip.env.SELECT_RAW end, {})
})
```

To use any `*SELECT*` variable, the `store_selection_keys` must be set via
`require("luasnip").config.setup({store_selection_keys="<Tab>"})`. In this case,
hitting `<Tab>` while in Visualmode will populate the `*SELECT*`-vars for the next
snippet and then clear them.


# LOADERS

Luasnip is capable of loading snippets from different formats, including both
the well-established vscode- and snipmate-format, as well as plain lua-files for
snippets written in lua

All loaders share a similar interface:
```lua
require("luasnip.loaders.from_{vscode,snipmate,lua}").{lazy_,}load(opts:table|nil)
```

where `opts` can contain the following keys:

- `paths`: List of paths to load. Can be a table, or a single
  comma-separated string.
  The paths may begin with `~/` or `./` to indicate that the path is
  relative to your `$HOME` or to the directory where your `$MYVIMRC` resides
  (useful to add your snippets).  
  If not set, `runtimepath` is searched for
  directories that contain snippets. This procedure differs slightly for
  each loader:
  - `lua`: the snippet-library has to be in a directory named
    `"luasnippets"`.
  - `snipmate`: similar to lua, but the directory has to be `"snippets"`.
  - `vscode`: any directory in `runtimepath` that contains a
    `package.json` contributing snippets.
- `exclude`: List of languages to exclude, empty by default.
- `include`: List of languages to include, includes everything by default.
- `{override,default}_priority`: These keys are passed straight to the
  [`add_snippets`](#api-reference)-calls and can therefore change the priority
  of snippets loaded from some colletion (or, in combination with
  `{in,ex}clude`, only some of its snippets).

While `load` will immediately load the snippets, `lazy_load` will defer loading until
the snippets are actually needed (whenever a new buffer is created, or the
filetype is changed luasnip actually loads `lazy_load`ed snippets for the
filetypes associated with this buffer. This association can be changed by
customizing `load_ft_func` in `setup`: the option takes a function that, passed
a `bufnr`, returns the filetypes that should be loaded (`fn(bufnr) -> filetypes
(string[])`)).

All of the loaders support reloading, so simply editing any file contributing
snippets will reload its snippets (only in the session the file was edited in,
we use `BufWritePost` for reloading, not some lower-level mechanism).

For easy editing of these files, Luasnip provides a [`vim.ui.select`-based
dialog](#edit_snippets) where first the filetype, and then the file can be
selected.

## Troubleshooting

* Luasnip uses `all` as the global filetype. As most snippet collections don't
  explicitly target luasnip, they may not provide global snippets for this
  filetype, but another, like `_` (`honza/vim-snippets`).
  In these cases, it's necessary to extend luasnip's global filetype with the
  collection's global filetype:
  ```lua
  ls.filetype_extend("all", { "_" })
  ```

  In general, if some snippets don't show up when loading a collection, a good
  first step is checking the filetype luasnip is actually looking into (print
  them for the current buffer via `:lua
  print(vim.inspect(require("luasnip").get_snippet_filetypes()))`), against the
  one the missing snippet is provided for (in the collection).  
  If there is indeed a mismatch, `filetype_extend` can be used to also search
  the collection's filetype:
  ```lua
  ls.filetype_extend("<luasnip-filetype>", { "<collection-filetype>" })
  ```

* As we only load `lazy_load`ed snippet on some events, `lazy_load` will
  probably not play nice when a non-default `ft_func` is used: if it depends on
  e.g. the cursor-position, only the filetypes for the cursor-position when the
  `lazy_load`-events are triggered will be loaded. Check
  [filetype_function's `extend_load_ft`](#filetype_functions) for a solution.

## VSCODE

As a reference on the structure of these snippet-libraries, see
[`friendly-snippets`](https://github.com/rafamadriz/friendly-snippets).

We support a small extension: snippets can contain luasnip-specific options in
the `luasnip`-table:
```json
"example1": {
	"prefix": "options",
	"body": [
		"whoa! :O"
	],
	"luasnip": {
		"priority": 2000,
		"autotrigger": true
	}
}
```

**Example**:

`~/.config/nvim/my_snippets/package.json`:
```json
{
	"name": "example-snippets",
	"contributes": {
		"snippets": [
			{
				"language": [
					"all"
				],
				"path": "./snippets/all.json"
			},
			{
				"language": [
					"lua"
				],
				"path": "./lua.json"
			}
		]
	}
}
```
`~/.config/nvim/my_snippets/snippets/all.json`:
```json
{
	"snip1": {
		"prefix": "all1",
		"body": [
			"expands? jumps? $1 $2 !"
		]
	},
	"snip2": {
		"prefix": "all2",
		"body": [
			"multi $1",
			"line $2",
			"snippet$0"
		]
	},
}
```

`~/.config/nvim/my_snippets/lua.json`:
```json
{
	"snip1": {
		"prefix": "lua",
		"body": [
			"lualualua"
		]
	}
}
```
This collection can be loaded with any of
```lua
-- don't pass any arguments, luasnip will find the collection because it is
-- (probably) in rtp.
require("luasnip.loaders.from_vscode").lazy_load()
-- specify the full path...
require("luasnip.loaders.from_vscode").lazy_load({paths = "~/.config/nvim/my_snippets"})
-- or relative to the directory of $MYVIMRC
require("luasnip.loaders.from_vscode").load({paths = "./my_snippets"})
```
## SNIPMATE

Luasnip does not support the full snipmate format: Only `./{ft}.snippets` and
`./{ft}/*.snippets` will be loaded. See
[honza/vim-snippets](https://github.com/honza/vim-snippets) for lots of
examples.

Like vscode, the snipmate-format is also extended to make use of some of
luasnips more advanced capabilities:
```snippets
priority 2000
autosnippet options
	whoa :O
```

**Example**:

`~/.config/nvim/snippets/c.snippets`:
```snippets
# this is a comment
snippet c c-snippet
	c!
```

`~/.config/nvim/snippets/cpp.snippets`:
```snippets
extends c

snippet cpp cpp-snippet
	cpp!
```

This can, again, be loaded with any of 
```lua
require("luasnip.loaders.from_snipmate").load()
-- specify the full path...
require("luasnip.loaders.from_snipmate").lazy_load({paths = "~/.config/nvim/snippets"})
-- or relative to the directory of $MYVIMRC
require("luasnip.loaders.from_snipmate").lazy_load({paths = "./snippets"})
```

Stuff to watch out for:

* Using both `extends <ft2>` in `<ft1>.snippets` and
  `ls.filetype_extend("<ft1>", {"<ft2>"})` leads to duplicate snippets.
* `${VISUAL}` will be replaced by `$TM_SELECTED_TEXT` to make the snippets
  compatible with luasnip
* We do not implement eval using \` (backtick). This may be implemented in the
  future.

## LUA

Instead of adding all snippets via `add_snippets`, it's possible to store them
in separate files and load all of those.
The file-structure here is exactly the supported snipmate-structure, e.g.
`<ft>.lua` or `<ft>/*.lua` to add snippets for the filetype `<ft>`.  
The files need to return two lists of snippets (either may be `nil`). The
snippets in the first are regular snippets for `<ft>`, the ones in the
second are autosnippets (make sure they are enabled in `setup` or `set_config`
if this table is used).

As defining all of the snippet-constructors (`s`, `c`, `t`, ...) in every file
is rather cumbersome, luasnip will bring some globals into scope for executing
these files.
By default, the names from [`luasnip.config.snip_env`][snip-env-src] will be used, but it's
possible to customize them by setting `snip_env` in `setup`.  

[snip-env-src]: https://github.com/L3MON4D3/LuaSnip/blob/69cb81cf7490666890545fef905d31a414edc15b/lua/luasnip/config.lua#L82-L104

**Example**:

`~/snippets/all.lua`:
```lua
return {
	parse("trig", "loaded!!")
}
```
`~/snippets/c.lua`:
```lua
return {
	parse("ctrig", "also loaded!!")
}, {
	parse("autotrig", "autotriggered, if enabled")
}
```

Load via 
```lua
require("luasnip.loaders.from_lua").load({paths = "~/snippets"})
```

## EDIT_SNIPPETS

To easily edit snippets for the current session, the files loaded by any loader
can be quickly edited via
`require("luasnip.loaders").edit_snippet_files(opts:table|nil)`  
When called, it will open a `vim.ui.select`-dialog to select first a filetype,
and then (if there are multiple) the associated file to edit.

`opts` currently only contains one setting:

* `format`: `fn(file:string, source_name:string) -> string|nil`  
  `file` is simply the path to the file, `source_name` is one of `"lua"`,
  `"snipmate"` or `"vscode"`.  
  If a string is returned, it is used as the title of the item, `nil` on the
  other hand will filter out this item.  
  The default simply replaces some long strings (packer-path and config-path)
  in `file` with shorter, symbolic names (`"$PLUGINS"`, `"$CONFIG"`), but
  this can be extended to
  * filter files from some specific source/path
  * more aggressively shorten paths using symbolic names, e.g.
  	`"$FRIENDLY_SNIPPETS"`

One comfortable way to call this function is registering it as a command:
```vim
command! LuaSnipEdit :lua require("luasnip.loaders").edit_snippet_files()
```

# SNIPPETPROXY

`SnippetProxy` is used internally to alleviate the upfront-cost of
loading snippets from e.g. a snipmate-library or a vscode-package. This is
achieved by only parsing the snippet on expansion, not immediately after reading
it from some file.
`SnippetProxy` may also be used from lua directly, to get the same benefits:

This will parse the snippet on startup...
```lua
ls.parser.parse_snippet("trig", "a snippet $1!")
```

... and this will parse the snippet upon expansion.
```lua
local sp = require("luasnip.nodes.snippetProxy")
sp("trig", "a snippet $1")
```


# EXT\_OPTS

`ext_opts` can be used to set the `opts` (see `nvim_buf_set_extmark`) of the
extmarks used for marking node-positions, either globally, per-snippet or
per-node.
This means that they allow highlighting the text inside of nodes, or adding
virtual text to the line the node begins on.

This is an example for the `node_ext_opts` used to set `ext_opts` of single nodes:
```lua
local ext_opts = {
	-- these ext_opts are applied when the node is active (e.g. it has been
	-- jumped into, and not out yet).
	active = 
	-- this is the table actually passed to `nvim_buf_set_extmark`.
	{
		-- highlight the text inside the node red.
		hl_group = "GruvboxRed"
	},
	-- these ext_opts are applied when the node is not active, but
	-- the snippet still is.
	passive = {
		-- add virtual text on the line of the node, behind all text.
		virt_text = {{"virtual text!!", "GruvboxBlue"}}
	},
	-- and these are applied when both the node and the snippet are inactive.
	snippet_passive = {}
}

...

s("trig", {
	i(1, "text1", {
		node_ext_opts = ext_opts
	}),
	i(2, "text2", {
		node_ext_opts = ext_opts
	})
})
```

In the above example the text inside the insertNodes is higlighted in red while
inside them, and the virtual text "virtual text!!" is visible as long as the
snippet is active.

It's important to note that `snippet_passive` applies to the states
`snippet_passive`, `passive`, and `active`, `passive` to `passive` and `active`,
and `active` only to `active`.

To disable a key from a "lower" state, it has to be explicitly set to its
default, e.g. to disable highlighting inherited from `passive` when the node is
`active`, `hl_group` could be set to `None` in `active`.

---

As stated earlier, these `ext_opts` can also be applied globally or for an
entire snippet. For this it's necessary to specify which kind of node a given
set of `ext_opts` should be applied to:

```lua
local types = require("luasnip.util.types")

ls.config.setup({
	ext_opts = {
		[types.insertNode] = {
			active = {...},
			passive = {...},
			snippet_passive = {...}
		},
		[types.choiceNode] = {
			active = {...}
		},
		[types.snippet] = {
			passive = {...}
		}
	}
})
```

The above applies the given `ext_opts` to all nodes of these types, in all
snippets...

```lua
local types = require("luasnip.util.types")

s("trig", { i(1, "text1"), i(2, "text2") }, {
	child_ext_opts = {
		[types.insertNode] = {
			passive = {
				hl_group = "GruvboxAqua"
			}
		}
	}
})
```
... while the `ext_opts` here are only applied to the `insertNodes` inside this
snippet.

---

By default, the `ext_opts` actually used for a node are created by extending the
`node_ext_opts` with the `effective_child_ext_opts[node.type]` of the parent,
which are in turn the `child_ext_opts` of the parent extended with the global
`ext_opts` set in the config.

It's possible to prevent both of these merges by passing
`merge_node/child_ext_opts=false` to the snippet/node-opts:

```lua
ls.config.setup({
	ext_opts = {
		[types.insertNode] = {
			active = {...}
		}
	}
})

...

s("trig", {
	i(1, "text1", {
		node_ext_opts = {
			active = {...}
		},
		merge_node_ext_opts = false
	}), i(2, "text2") }, {
	child_ext_opts = {
		[types.insertNode] = {
			passive = {...}
		}
	},
	merge_child_ext_opts = false
})
```

---

The `hl_group` of the global `ext_opts` can also be set via standard
highlight-groups:

```lua
vim.cmd("hi link LuasnipInsertNodePassive GruvboxRed")
vim.cmd("hi link LuasnipSnippetPassive GruvboxBlue")

-- needs to be called for resolving the actual ext_opts.
ls.config.setup({})
```
The names for the used highlight groups are
`"Luasnip<node>{Passive,Active,SnippetPassive}"`, where `<node>` can be any kind of
node in PascalCase (or "Snippet").

---

One problem that might arise when nested nodes are highlighted, is that the
highlight of inner nodes should be visible above that of nodes they are nested inside.

This can be controlled using the `priority`-key in `ext_opts`. Normally, that
value is an absolute value, but here it is relative to some base-priority, which
is increased for each nesting level of snippets.

Both the initial base-priority and its' increase and can be controlled using
`ext_base_prio` and `ext_prio_increase`:
```lua
ls.config.setup({
	ext_opts = {
		[types.insertNode] = {
			active = {
				hl_group = "GruvboxBlue",
				-- the priorities should be \in [0, ext_prio_increase).
				priority = 1
			}
		},
		[types.choiceNode] = {
			active = {
				hl_group = "GruvboxRed"
				-- priority defaults to 0
			}
		}
	}
	ext_base_prio = 200,
	ext_prio_increase = 2
})
```
Here the highlight of an insertNode nested directly inside a choiceNode is
always visible on top of it.


# DOCSTRING

Snippet-docstrings can be queried using `snippet:get_docstring()`. The function
evaluates the snippet as if it was expanded regularly, which can be problematic
if e.g. a dynamicNode in the snippet relies on inputs other than
the argument-nodes.
`snip.env` and `snip.captures` are populated with the names of the queried
variable and the index of the capture respectively
(`snip.env.TM_SELECTED_TEXT` -> `'$TM_SELECTED_TEXT'`, `snip.captures[1]` ->
 `'$CAPTURES1'`). Although this leads to more expressive docstrings, it can
 cause errors in functions that e.g. rely on a capture being a number:

```lua
s({trig = "(%d)", regTrig = true}, {
	f(function(args, snip)
		return string.rep("repeatme ", tonumber(snip.captures[1]))
	end, {})
}),
```

This snippet works fine because	`snippet.captures[1]` is always a number.
During docstring-generation, however, `snippet.captures[1]` is `'$CAPTURES1'`,
which will cause an error in the functionNode.
Issues with `snippet.captures` can be prevented by specifying `docTrig` during
snippet-definition:

```lua
s({trig = "(%d)", regTrig = true, docTrig = "3"}, {
	f(function(args, snip)
		return string.rep("repeatme ", tonumber(snip.captures[1]))
	end, {})
}),
```

`snippet.captures` and `snippet.trigger` will be populated as if actually
triggered with `3`.

Other issues will have to be handled manually by checking the contents of e.g.
`snip.env` or predefining the docstring for the snippet:

```lua
s({trig = "(%d)", regTrig = true, docstring = "repeatmerepeatmerepeatme"}, {
	f(function(args, snip)
		return string.rep("repeatme ", tonumber(snip.captures[1]))
	end, {})
}),
```



# DOCSTRING-CACHE

Although generation of docstrings is pretty fast, it's preferable to not
redo it as long as the snippets haven't changed. Using
`ls.store_snippet_docstrings(snippets)` and its counterpart
`ls.load_snippet_docstrings(snippets)`, they may be serialized from or
deserialized into the snippets.
Both functions accept a table structsured like this: `{ft1={snippets},
ft2={snippets}}`. Such a table containing all snippets can be obtained via
`ls.get_snippets()`.
`load` should be called before any of the `loader`-functions as snippets loaded
from vscode-style packages already have their `docstring` set (`docstrings`
wouldn't be overwritten, but there'd be unnecessary calls).

The cache is located at `stdpath("cache")/luasnip/docstrings.json` (probably
`~/.cache/nvim/luasnip/docstrings.json`).

# EVENTS

Events can be used to react to some action inside snippets. These callbacks can
be defined per-snippet (`callbacks`-key in snippet constructor) or globally
(autocommand).

`callbacks`: `fn(node[, event_args]) -> event_res`  
All callbacks get the `node` associated with the event and event-specific
optional arguments, `event_args`.
`event_res` is only used in one event, `pre_expand`, where some properties of
the snippet can be changed.

`autocommand`:
Luasnip uses `User`-events. Autocommands for these can be registered using
```vim
au User SomeUserEvent echom "SomeUserEvent was triggered"
```

or
```lua
vim.api.nvim_create_autocommand("User", {
	patter = "SomeUserEvent",
	command = "echom SomeUserEvent was triggered"
})
```
The node and `event_args` can be accessed through `require("luasnip").session`:

* `node`: `session.event_node`  
* `event_args`: `session.event_args`

**Events**:

* `enter/leave`: Called when a node is entered/left (for example when jumping
  around in a snippet).  
  `User-event`: `"Luasnip<Node>{Enter,Leave}"`, with `<Node>` in
  PascalCase, e.g. `InsertNode` or `DynamicNode`.  
  `event_args`: none
* `change_choice`: When the active choice in a choiceNode is changed.  
  `User-event`: `"LuasnipChangeChoice"`  
  `event_args`: none
* `pre_expand`: Called before a snippet is expanded. Modifying text is allowed,
  the expand-position will be adjusted so the snippet expands at the same
  position relative to existing text.  
  `User-event`: `"LuasnipPreExpand"`  
  `event_args`:
  * `expand_pos`: `{<row>, <column>}`, position at which the snippet will be
  	expanded. `<row>` and `<column>` are both 0-indexed.
  `event_res`:
  * `env_override`: `map string->(string[]|string)`, override or extend the
    snippet's environment (`snip.env`).

A pretty useless, beyond serving as an example here, application of these would
be printing e.g. the nodes' text after entering:

```lua
vim.api.nvim_create_autocmd("User", {
	pattern = "LuasnipInsertNodeEnter",
	callback = function()
		local node = require("luasnip").session.event_node
		print(table.concat(node:get_text(), "\n"))
	end
})
```

or some information about expansions

```lua
vim.api.nvim_create_autocmd("User", {
	pattern = "LuasnipPreExpand",
	callback = function()
		-- get event-parameters from `session`.
		local snippet = require("luasnip").session.event_node
		local expand_position =
			require("luasnip").session.event_args.expand_pos

		print(string.format("expanding snippet %s at %s:%s",
			table.concat(snippet:get_docstring(), "\n"),
			expand_position[1],
			expand_position[2]
		))
	end
})
```

# CLEANUP
The function ls.cleanup()  triggers the `LuasnipCleanup` user-event, that you can listen to do some kind
of cleaning in your own snippets, by default it will  empty the snippets table and the caches of
the lazy_load.

# API-REFERENCE

`require("luasnip")`:

- `add_snippets(ft:string or nil, snippets:list or table, opts:table or nil)`:
  Makes `snippets` (list of snippets) available in `ft`.  
  If `ft` is `nil`, `snippets` should be a table containing lists of snippets,
  the keys are corresponding filetypes.  
  `opts` may contain the following keys:
  - `type`: type of `snippets`, `"snippets"` or `"autosnippets"`.
  - `key`: Key that identifies snippets added via this call.  
	If `add_snippets` is called with a key that was already used, the snippets
	from that previous call will be removed.  
	This can be used to reload snippets: pass an unique key to each
	`add_snippets` and just re-do the `add_snippets`-call when the snippets have
	changed.
  - `override_priority`: set priority for all snippets.
  - `default_priority`: set priority only for snippets without snippet-priority.

- `clean_invalidated(opts: table or nil) -> bool`: clean invalidated snippets
  from internal snippet storage.  
  Invalidated snippets are still stored, it might be useful to actually remove
  them, as they still have to be iterated during expansion.

  `opts` may contain:

  - `inv_limit`: how many invalidated snippets are allowed. If the number of
  	invalid snippets doesn't exceed this threshold, they are not yet cleaned up.

	A small number of invalidated snippets (<100) probably doesn't affect
	runtime at all, whereas recreating the internal snippet storage might.

- `get_id_snippet(id)`: returns snippet corresponding to id.

- `in_snippet()`: returns true if the cursor is inside the current snippet.

- `jumpable(direction)`: returns true if the current node has a
  next(`direction` = 1) or previous(`direction` = -1), e.g. whether it's
  possible to jump forward or backward to another node.

- `jump(direction)`: returns true if the jump was successful.

- `expandable()`: true if a snippet can be expanded at the current cursor position.

- `expand()`: expands the snippet at(before) the cursor.

- `expand_or_jumpable()`: returns `expandable() or jumpable(1)` (exists only
  because commonly, one key is used to both jump forward and expand).

- `expand_or_locally_jumpable()`: same as `expand_or_jumpable()` except jumpable
  is ignored if the cursor is not inside the current snippet.

- `expand_or_jump()`: returns true if jump/expand was succesful.

- `expand_auto()`: expands the autosnippets before the cursor (not necessary
  to call manually, will be called via autocmd if `enable_autosnippets` is set
  in the config).

- `snip_expand(snip, opts)`: expand `snip` at the current cursor position.
  `opts` may contain the following keys:
    - `clear_region`: A region of text to clear after expanding (but before
      jumping into) snip. It has to be at this point (and therefore passed to
      this function) as clearing before expansion will populate `TM_CURRENT_LINE`
      and `TM_CURRENT_WORD` with wrong values (they would miss the snippet trigger)
      and clearing after expansion may move the text currently under the cursor
      and have it end up not at the `i(1)`, but a `#trigger` chars to it's right.
      The actual values used for clearing are `from` and `to`, both (0,0)-indexed
      byte-positions.
      If the variables don't have to be populated with the correct values, it's
      safe to remove the text manually.
    - `expand_params`: table, for overriding the `trigger` used in the snippet
      and setting the `captures` (useful for pattern-triggered nodes where the
      trigger has to be changed from the pattern to the actual text triggering the
      node).
      Pass as `trigger` and `captures`.
    - `pos`: position (`{line, col}`), (0,0)-indexed (in bytes, as returned by
      `nvim_win_get_cursor()`), where the snippet should be expanded. The
      snippet will be put between `(line,col-1)` and `(line,col)`. The snippet
      will be expanded at the current cursor if pos is nil.
    - `jump_into_func`: fn(snippet) -> node:
      Callback responsible for jumping into the snippet. The returned node is
      set as the new active node, ie. it is the origin of the next jump.
      The default is basically this
      ```lua
      function(snip)
      	-- jump_into set the placeholder of the snippet, 1
      	-- to jump forwards.
      	return snip:jump_into(1)
      ```
      While this can be used to only insert the snippet
      ```lua
      function(snip)
      	return snip.insert_nodes[0]
      end
      ```

  `opts` and any of its parameters may be nil.

- `get_active_snip()`: returns the currently active snippet (not node!).

- `choice_active()`: true if inside a choiceNode.

- `change_choice(direction)`: changes the choice in the innermost currently
  active choiceNode forward (`direction` = 1) or backward (`direction` = -1).

- `unlink_current()`: removes the current snippet from the jumplist (useful
  if luasnip fails to automatically detect e.g. deletion of a snippet) and
  sets the current node behind the snippet, or, if not possible, before it.

- `lsp_expand(snip_string, opts)`: expand the lsp-syntax-snippet defined via
  `snip_string` at the cursor.
  `opts` can have the same options as `opts` in `snip_expand`.

- `active_update_dependents()`: update all function/dynamicNodes that have the
  current node as an argnode (will actually only update them if the text in any
  of the argnodes changed).

- `available()`: return a table of all snippets defined for the current
  filetypes(s) (`{ft1={snip1, snip2}, ft2={snip3, snip4}}`).

- `exit_out_of_region(node)`: checks whether the cursor is still within the
  range of the snippet `node` belongs to. If yes, no change occurs, if No, the
  snippet is exited and following snippets' regions are checked and potentially
  exited (the next active node will be the 0-node of the snippet before the one
  the cursor is inside.
  If the cursor isn't inside any snippet, the active node will be the last node
  in the jumplist).
  If a jump causes an error (happens mostly because a snippet was deleted), the
  snippet is removed from the jumplist.

- `store_snippet_docstrings(snippet_table)`: Stores the docstrings of all
  snippets in `snippet_table` to a file
  (`stdpath("cache")/luasnip/docstrings.json`).
  Calling `store_snippet_docstrings(snippet_table)` after adding/modifying
  snippets and `load_snippet_docstrings(snippet_table)` on startup after all
  snippets have been added to `snippet_table` is a way to avoide regenerating
  the (unchanged) docstrings on each startup.
  (Depending on when the docstrings are required and how luasnip is loaded,
  it may be more sensible to let them load lazily, e.g. just before they are
  required).
  `snippet_table` should be laid out just like `luasnip.snippets` (it will
  most likely always _be_ `luasnip.snippets`).

- `load_snippet_docstrings(snippet_table)`: Load docstrings for all snippets
  in `snippet_table` from `stdpath("cache")/luasnip/docstrings.json`.
  The docstrings are stored and restored via trigger, meaning if two
  snippets for one filetype have the same(very unlikely to happen in actual
  usage), bugs could occur.
  `snippet_table` should be laid out as described in `store_snippet_docstrings`.

- `unlink_current_if_deleted()`: Checks if the current snippet was deleted,
  if so, it is removed from the jumplist. This is not 100% reliable as
  luasnip only sees the extmarks and their beginning/end may not be on the same
  position, even if all the text between them was deleted.

- `filetype_extend(filetype:string, extend_filetypes:table of string)`: Tells
  luasnip that for a buffer with `ft=filetype`, snippets from
  `extend_filetypes` should be searched as well. `extend_filetypes` is a
  lua-array (`{ft1, ft2, ft3}`).
  `luasnip.filetype_extend("lua", {"c", "cpp"})` would search and expand c-and
  cpp-snippets for lua-files.

- `filetype_set(filetype:string, replace_filetypes:table of string):` Similar
  to `filetype_extend`, but where _append_ appended filetypes, _set_ sets them:
  `filetype_set("lua", {"c"})` causes only c-snippets to be expanded in
  lua-files, lua-snippets aren't even searched.

- `cleanup()`: clears all snippets. Not useful for regular usage, only when
  authoring and testing snippets.

- `refresh_notify(ft:string)`: Triggers an autocmd that other plugins can hook
  into to perform various cleanup for the refreshed filetype.
  Useful for signaling that new snippets were added for the filetype `ft`.
- `set_choice(indx:number)`: Changes to the `indx`th choice.
  If no `choiceNode` is active, an error is thrown.
  If the active `choiceNode` doesn't have an `indx`th choice, an error is
  thrown.
- `get_current_choices() -> string[]`: Returns a list of multiline-strings
  (themselves lists, even if they have only one line), the `i`th string
  corresponding to the `i`th choice of the currently active `choiceNode`.
  If no `choiceNode` is active, an error is thrown.

Not covered in this section are the various node-constructors exposed by
the module, their usage is shown either previously in this file or in
`Examples/snippets.lua` (in the repo).
