+++
title = "Code Folding in Neovim with Tree-sitter"
date = 2022-03-09T17:19:12-05:00
author = "John Maguire"
cover = ""
tags = ["Neovim", "Tree-sitter", "Vim", "Lua", "Code Folding"]
keywords = ["Neovim", "Tree-sitter", "Vim", "Code Folding", "Lua"]
description = "Quick guide on enabling Tree-sitter powered code folding"
showFullContent = false
+++


{{<figure
    src="/img/code_folding.png"
    alt="Code folding in Neovim"
    position="center"
    style=""
    caption="Folding some go code (theme is <a href='https://github.com/sainnhe/everforest'>everforest</a>)"
    captionPosition="center"
    captionStyle="background:none; color: white; font-style: italic"
>}}

Code folding can be a really useful editor feature to pare down the visible code on screen in a large file to only the
bits you need to work. With Neovim we've got a few options on how to handle what gets folded when we create a fold, and
in this post we're going to do a quick overview of how to enable syntax based code folding powered by Tree-sitter. Though
before we dive in, a quick background on Tree-sitter: it's a parser generator tool that uses the grammar for a language
to develop the parser. One of the main uses of it is to enable more intelligent syntax highlighting in your editor,
but as you'll see here it can also do a lot more than just highlighting. If you want to read more check out their
[documentation](https://tree-sitter.github.io/tree-sitter/).


Now, let's modify our Neovim setup to enable code folding and have Tree-sitter do the heavy lifting (one note, you must
be using Neovim 0.5+ to have Tree-sitter be supported). The only thing we need to do is enable folding and tell neovim
to use Tree-sitter:

_vimscript_:
```vim
set foldmethod=expr
set foldexpr=nvim_treesitter#foldexpr()
```

_lua_:
```lua
local vim = vim
local opt = vim.opt

opt.foldmethod = "expr"
opt.foldexpr = "nvim_treesitter#foldexpr()"
```


That will get you all setup! Easy right? Now the one thing I found to be annoying was that this will default all folds
to be closed on any file that you open. To switch that, so all folds are open on any file you open, we're going to add an
autocommand to run after a file/buffer is opened to open all folds in that file:

_vimscript_:
```vim
autocmd BufReadPost,FileReadPost * normal zR
```

_lua_:
```lua
local vim = vim
local api = vim.api
local M = {}
-- function to create a list of commands and convert them to autocommands
-------- This function is taken from https://github.com/norcalli/nvim_utils
function M.nvim_create_augroups(definitions)
    for group_name, definition in pairs(definitions) do
        api.nvim_command('augroup '..group_name)
        api.nvim_command('autocmd!')
        for _, def in ipairs(definition) do
            local command = table.concat(vim.tbl_flatten{'autocmd', def}, ' ')
            api.nvim_command(command)
        end
        api.nvim_command('augroup END')
    end
end


local autoCommands = {
    -- other autocommands
    open_folds = {
        {"BufReadPost,FileReadPost", "*", "normal zR"}
    }
}

M.nvim_create_augroups(autoCommands)
```

So there are a few things to point out with the above code blocks, the first being the `normal zR` part. `zR` is a neovim
command to open _all_ folds in a file and `normal` tells neovim to execute that command in normal mode (for some more
reading on the different folding commands check out the [Neovim documentation on fold commands](https://neovim.io/doc/user/fold.html#fold-commands)).
The one other bit to point out is the large block of code for the lua setup. As of writing this Neovim does not support
defining autocommands in lua, though support for that should be dropping with Neovim 0.7. For now though, we use a
separate function that takes in a list of strings defining the autocommands we want and sets them using the
`nvim_command` api.

And that's it! Now you've got code folding based on syntax powered by Tree-sitter. You can check out the rest of my Neovim
setup [here](https://github.com/jm96441n/dotfiles/tree/master/.config/nvim).


