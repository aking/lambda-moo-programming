# Code Scanner
This package will scan verb code for common issues and warn you about them. You can edit/add/remove some checks if you don't want them, or update them to better fit your coding standards.

## MOO Package Manager

THIS PACKAGE IS AVAILABLE FOR INSTALL USING THE MOO PACKAGE MANAGER (FOR TOASTSTUNT). THIS IS THE PREFERRED METHOD OF INSTALL IF YOU ARE USING TOASTSTUNT. IT IS PROBABLY A MORE UPDATED VERSION AS WELL. SEE: [MOO Package Manager](https://github.com/sevenecks/moo-package-manager)

## Contributing

The code scanner covers some of our best practices for MOO coding on [Sindome](https://www.sindome.org/). If you have your own that you think would be useful to the community please feel free to put in a pull request!

## Requirements
This code was tested on stock LambdaMOO 1.8.1 running the lastest LambdaCore.db. AND on ToastStunt running a modified LambdaCore.db. If your DB is based off of that it should work with no changes needed.

## What's Scanned For?

1. Comment at the top of the verb for non-player facing verbs (good practice!)
2. Nesting of for/if/while > MAX_NESTING (a variable you can define in your code).
3. Argument scatter in player facing verbs
4. Object Numbers in code (you should corify things for clarity (and later your own sanity))
5. tostr usage inside a :tell (this is already tostr'd on #1)
6. .location assignment instead of comparison (so you don't loop over a bunch of objects and move them instead of checking their location)
7. assignment in if statements (doing assignment in if statements is valid in moo, and useful, but it's good to be warned of in general so you don't assign instead of compare by accident)
8. forking (forks are useful but writing a $scheduler that lets you $schedule things to be run at specific times is better)
9. verb length > MAX_LENGTH (a variable you can change to suit your needs)

## Installation
Create an object, we're using #79 as the parent, but it really doesn't rely on the parent anything but itself, so use your own discresion.

```
@create #78 named Code Scanner
```

Now you'll want to corify the reference to the Code Scanner, replace #97 with your new object number.

```
@corify #97 as code_scanner
```

This allows you to reference the object as $code_scanner.

Now, copy the code below into a text editor and change the obj# form #97 to whatever the obj# of your newly created Code Scanner object.
```
;#97.description = {"MOO Code Scanner 1.1 by Brendan Butts <slither@sindome.org>", "", "Github: https://github.com/SevenEcks/lambda-moo-programming", "", "Usage: $code_scanner:scan_for_issues(OBJ, verbname)", "Usage: $code_scanner:display_issues($code_scanner:scan_for_issues(OBJ, verbname))", "", "If you integrate this with your @Program verb I recommend you make a copy of it first and test on that just in case! But you can always use .program if you mess up!"}
@create #12654 named Code Scanner Utils:Code,Scanner,Utils,Code Scanner Utils
;;#97.("aliases") = {"Code", "Scanner", "Utils", "Code Scanner Utils"}
;;#97.("description") = "This is a placeholder parent for all the $..._utils packages, to more easily find them and manipulate them. At present this object defines no useful verbs or properties. (Filfre.)"
;;#97.("object_size") = {19212, 1612285023}
;;#97.("instance_id") = "#12654-1541212667.02364"

@verb #97:"scan_for_issues" this none this
@program #97:scan_for_issues
{object, verbname, ?options = {}, ?code = {}} = args;
if (!code)
  code = verb_code(object, verbname);
endif
verb_args = verb_args(object, verbname);
"max length before we start warning it's too long";
MAX_LENGTH_WARNING = 40;
"max nesting before we start warning";
MAX_NESTING_WARNING = 2;
"what do the args of an internal variable look like?";
internal_args = {"this", "none", "this"};
warnings = {};
max_nest = 0;
open_ifs = 0;
open_fors = 0;
open_whiles = 0;
forks = 0;
"first real line of code, not a comment";
first_real_line = 0;
"count of what line we are on";
count = 0;
"is this an internal (tnt) verb?";
internal = (internal_args == verb_args) ? 1 | 0;
for line in (code)
  count = count + 1;
  "check for opening comments";
  if (((count == 1) && internal) && (!$code_scanner:match_comment(line)))
    warnings = {@warnings, {"You did not include a comment on the first line describing the use and args of your verb.", count}};
  endif
  "check if we have found the first real line of code or if it's just a comment";
  if ((!first_real_line) && (!$code_scanner:match_comment(line)))
    first_real_line = count;
  endif
  "check for an argument scatter in a player facing verb";
  if ((!internal) && $code_scanner:match_arg_scatter(line))
    warnings = {@warnings, {"You have used an argument scatter in a verb that is not 'this none this'. This is not how it should work.", count}};
  endif
  "find nesting";
  if ($code_scanner:match_if(line))
    open_ifs = open_ifs + 1;
  elseif ($code_scanner:match_for(line))
    open_fors = open_fors + 1;
  elseif ($code_scanner:match_while(line))
    open_whiles = open_whiles + 1;
  elseif ($code_scanner:match_endif(line))
    open_ifs = open_ifs - 1;
  elseif ($code_scanner:match_endfor(line))
    open_fors = open_fors - 1;
  elseif ($code_scanner:match_endwhile(line))
    open_whiles = open_whiles - 1;
  elseif ($code_scanner:match_fork(line))
    forks = forks + 1;
  elseif ($code_scanner:match_object(line))
    warnings = {@warnings, {"You may have an object number in your code. This should be avoided. Use corified references ($) instead.", count}};
  endif
  "check for tostr usage in :tell verbs";
  if ($code_scanner:match_tell_tostr(line))
    warnings = {@warnings, {"tostr usage found inside a :tell call. This is undeeded as :tell will call tostr on all it's args.", count}};
  endif
  if ($code_scanner:match_location_assignment(line))
    warnings = {@warnings, {"IMPORTANT!!!!!!!!!!!!!!!! You may have included a .location assignment INSTEAD of a comparison (= instead of ==)!", count}};
  endif
  if ($code_scanner:match_if_assignment(line))
    warnings = {@warnings, {"You are doing an assignment '=' operation in an if statement, please confirm you didn't mean to do an equality check '=='.", count}};
  endif
  if ($code_scanner:match_recycler_valid(line))
    warnings = {@warnings, {"You are doing an if($recycler:valid()) operation without a ! in front of it, are you SURE that you don't need the bang (!)?", count}};
  endif
  if ((current_nest = (open_fors + open_ifs) + open_whiles) > max_nest)
    max_nest = current_nest;
  endif
endfor
if (forks)
  warnings = {@warnings, {"There is a fork() in this code. Please do not do this unless you know what you are doing. Consider using the $scheduler (or something with a heartbeat) instead.", 0}};
endif
if (max_nest > MAX_NESTING_WARNING)
  warnings = {@warnings, {tostr("Max nesting of if/for/while's is ", max_nest, ". Try refactoring or extracting pieces to a new verb to get your max nesting to 2 or below."), 0}};
endif
length_of_verb = length(code);
if (length_of_verb > MAX_LENGTH_WARNING)
  warnings = {@warnings, {tostr("This verb is ", length_of_verb, " lines long. Consider refactoring or extracting to a new verb to get your max nesting to ", MAX_LENGTH_WARNING, " or below."), 0}};
endif
return warnings;
.

@verb #97:"display_issues" this none this
@program #97:display_issues
":display_issues(LIST warnings) => none";
"takes the output of :scan_for_issues and displays it";
{warnings} = args;
for warning_set in (warnings)
  {warning, line_number} = warning_set;
  if (line_number)
    player:tell(tostr("Warning on line ", line_number), ": ", warning);
  else
    player:tell("Warning", ": ", warning);
  endif
endfor
"VMS NOTE display messages #22664 07/17/18  1:16";
"VMS VERSION 1.01";
"Last modified by Fengshui (#22664) on Tue Jul 17 13:16:35 2018 PDT";
.

@verb #97:"match_if" this none this
@program #97:match_if
":match_if(STR line) => bool";
{line} = args;
return match(line, "^[ ]*if ");
.

@verb #97:"match_for" this none this
@program #97:match_for
":match_for(STR line) => bool";
{line} = args;
return match(line, "^[ ]*for ");
.

@verb #97:"match_while" this none this
@program #97:match_while
":match_while(STR line) => bool";
{line} = args;
return match(line, "^[ ]*while ");
.

@verb #97:"match_endif" this none this
@program #97:match_endif
":match_endif(STR line) => bool";
{line} = args;
return match(line, "^[ ]*endif$");
.

@verb #97:"match_endfor" this none this
@program #97:match_endfor
":match_if(STR line) => bool";
{line} = args;
return match(line, "^[ ]*endfor$");
.

@verb #97:"match_endwhile" this none this
@program #97:match_endwhile
":match_endwhile(STR line) => bool";
{line} = args;
return match(line, "^[ ]*endwhile");
.

@verb #97:"match_fork" this none this
@program #97:match_fork
":match_fork(STR line) => bool";
{line} = args;
return match(line, "^[ ]*fork");
.

@verb #97:"match_object" this none this
@program #97:match_object
":match_object(STR line) => bool";
{line} = args;
return match(line, "#[0-9]+");
.

@verb #97:"match_tell_tostr" this none this
@program #97:match_tell_tostr
":match_tell_tostr(STR line) => bool";
{line} = args;
return match(line, ".*:tell(.*tostr(");
.

@verb #97:"match_comment" this none this
@program #97:match_comment
":match_comment(STR line) => bool";
{line} = args;
return match(line, "^[ ]*\"");
.

@verb #97:"match_arg_scatter" this none this
@program #97:match_arg_scatter
":match_arg_scatter(STR line) => bool";
{line} = args;
return match(line, "{.+} = args;");
.

@verb #97:"match_location_assignment" this none this
@program #97:match_location_assignment
":match_if(STR line) => bool";
{line} = args;
return match(line, "[ ]*if (.+.location = .+)");
.

@verb #97:"match_if_assignment" this none this
@program #97:match_if_assignment
":match_if_assignment(STR line) => bool";
"looks for assignment operators in if statements";
{line} = args;
return match(line, "^[ ]*if (.+ = .+)");
.

@verb #97:"match_corified_verb_anywhere" this none this
@program #97:match_corified_verb_anywhere
":match_corified(STR line) => bool";
"this matches to a corified verb reference ANYWHERE in the string";
{line} = args;
return match(line, "%$[a-z]+:[a-z]+[^(]");
.

@verb #97:"match_recycler_valid" this none this
@program #97:match_recycler_valid
":match_recycler_valid(STR line) => bool";
{line} = args;
return match(line, "^[ ]*if (%$recycler:valid");
.
```

## Usage

```
;$code_scanner:display_issues($code_scanner:scan_for_issues($code_scanner, "scan_for_issues"))
```

## Colorizing
If you have ANSI support you can edit :display_issues to add some colorizing.

## Integrating with @program
It's great to have these warnings show up automatically after you @program a verb. However you must be careful when integrating the code scanner as you don't want to break your @program verb to the point where you can't @program a fix for it. 

I recommend two things:

1. Create a copy of your @program verb and test integration with that.
2. Put the $code_scanner code inside a try/except just to be on the safe side.

### Where do I put the code inside @program?

Put it somewhere after the set_verb_code() builtin is executed.

## Forks

Sindome has a $scheduler which is an alternative to forking, that lets you schedule things to happen at specific times, as opposed to forking in your code. Other MOO's have similar things. If you do not, you can remove the fork() check, if you don't find it useful.
