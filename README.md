make4ht
=======

Simple build system for tex4ht. Some day it may replace `t4ht` for generating `css` and `images`. First aim is to copy or move generated files to correct directories and possibility to write build scripts, allowing to run bibtex or indexing programs from make4ht run.

How it works
------------

Default compilation scripts for `tex4ht` compiles LaTeX files with this sequence:

    latex $latex_options 'code for loading tex4ht.sty \input{filename}'
    latex $latex_options 'code for loading tex4ht.sty \input{filename}'
    latex $latex_options 'code for loading tex4ht.sty \input{filename}'
    tex4ht options filename
    t4ht options filename

Problem is, if you need to run indexing program or bibtex, you need to create new script based on old one, or run htlatex twice, which means six LaTeX runs.

Other problem is with `t4ht` application. It reads file `filename.lg`, generated by `tex4ht`, where are instructions about generated files, css instructions, calls to external applications, instructions for image conversions etc. It can be instrued to copy generated files to some output directory, but it doesn't preserve directory structure, which means that if you have images in some subdirectory, they will be copied to output directory, but links will be pointing to non existing subdir.

Image conversion is directed with the [env file](http://www.tug.org/applications/tex4ht/mn35.html#index35-73001), with really strange syntax based on whitespace and [os dependent](http://www.tug.org/applications/tex4ht/mn-unix.html#index27-69005). Build scripts should be able to take image conversion process and also to enable actions like conversion with xslt processor ot tidy on output files, based on file extensions.

Idea is to use LUA buildfile, with name `filename + .mk4 extension` and specify there actions.

Build file
----------

Sample:

    Make:htlatex()
    Make:match("html$", "tidy -m -xml -utf8 -q -i ${filename}")

This build file will run htlatex one time. You can add more commands like `Make:htlatex` with 

    Make:add("name", "command", {default parameters})

you can run then 

    Make:name()

`command` can be text template, or function:

    Make:add("text", "hello, input file: ${input}")
    Make:add("function", function(params) for k, v in pairs(params) do print(k..": "..v) end)

Default parameters are:

  - htlatex - used compiler
  - input - input file
  - latex\_par - parameters to latex
  - tex4ht\_sty\_par - parameters to tex4ht.sty
  - tex4ht\_par - parameters to tex4ht application
  - t4ht\_par - parameters to t4ht application
  - outdir - output directory


Other type of actions which can be specified in build file are
functions which are running on the generated files:

    Make:match("html$", "tidy -m -xml -utf8 -q -i ${filename}")

This tests filenames with lua pattern matching and on matched items it run 
command or function specified in second argument". Parameters are the same, as in previous section, except filename, which is generated output name.

### Filters

You can use filter module to modify contents of generated html files. 
This is useful for fixing some tex4ht bugs.

Example:

    local filter = require "make4ht-filter"
    local process = filter{"cleanspan", "fixligatures", "hruletohr"}
    Make:htlatex()
    Make:htlatex()
    Make:match("html$",process)


Filter module is located in `make4ht-filter`. Function is returned, 
which is used for building filter chains then. 

Built-in filters are:

 - cleanspan - clean spurious span elements when accented characters are used
 - fixligatures - decompose ligatures to base characters
 - hruletohr - \hrule commands are translated to series of underscore characters
   by `tex4ht`, this filter translate these underscores to `<hr>` elements

Function `filter` accepts also function arguments, in this case this function 
takes file contents as parameter and modified contents are returned.

Example:

    local filter  = require "make4ht-filter"                                    
    local changea = function(s) return s:gsub("a","z") end
    local process = filter{"cleanspan", "fixligatures", changea}            
    Make:htlatex()                                                              
    Make:htlatex()                                                                  Make:match("html$",process) 

In this case, spurious span elements are joined, ligatures are decomposed,
and then all letters 'a' are replaced with 'z' letters.

### `mode` variable

Variable `mode` contains contents of `-mode` command line option. 
It can be used to run some commands conditionally. For example:

     if mode == "draft" then
       Make:htlatex{} 
     else
       Make:htlatex{}
       Make:htlatex{}
       Make:htlatex{}
     end

In this example (which is default configuration used by `make4ht`),
LaTeX is called only once when `make4ht` is called with
    
    make4ht -m draft filename

Instalation
-----------

Find your local texmf tree with command:

    kpsewhich -var-value TEXMFHOME

and go to directory `scripts/lua` 
(you will need to create it, if it doesn't exist).
In this directory, run:

    git clone git://github.com/michal-h21/make4ht.git

Command line options
--------------------

    make4ht - build system for tex4ht
    Usage:
    make4ht [options] filename ["tex4ht.sty op." "tex4ht op." "t4ht op" "latex op"]
    -c,--config (default xhtml) Custom config file
    -d,--output-dir (default "")  Output directory
    -l,--lua  Use lualatex for document compilation
    -m,--mode (default default) Switch which can be used in the makefile
    -s,--shell-escape Enables running external programs from LaTeX
    -u,--utf8  For output documents in utf8 encoding
    -x,--xetex Use xelatex for document compilation
    <filename> (string) Input file name


You can still use `make4ht` in same way as `htlatex`

    make4ht filename "customcfg, charset=utf-8" " -cunihtf -utf8" " -dfoo"

but note that this will not use `make4ht` routines for output dir making and copying. If you want to use them, change the line above to:

    make4ht filename "customcfg, charset=utf-8" " -cunihtf -utf8"  -d foo

this is the same as:

    make4ht -u -c customcfg -d foo filename

Output directory doesn't have to exist, it will be created automaticaly. 
Specified path can be relative to current directory, or absolute:

    make4ht -d use/current/dir/ filename
    make4ht -d ../gotoparrentdir filename
    make4ht -d ~/gotohomedir filename
    make4ht -d c:\documents\windowspathsareworkingtoo filename


Future plans
------------

  - extend documentation
  - add commands for image conversion

