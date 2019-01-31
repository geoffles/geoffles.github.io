---
layout: default
title: Progress Indicators in your Bash Scripts
categories: Development
tags: [bash]
date: 2019-01-31 07:58:00+0200
---

***Abstract:** Adding a progress indicators to shell scripts, including a count and total of completed, percentage complete, a progress bar, and time remaining.*

# Introduction

I've often built some long running shell scripts. These usually do some kind of ad hoc data extraction, transformation, or just collecting a bunch of things from a long list. In these cases, it's often nice to be able see how far you are and maybe to have an idea of how long it has left to finish the work.

This post covers progress reporting for Bash scripts.

# The Basics

Progress reporting is achieved by rewriting the same line on the output, which can be achieved with a carriage return character `\r` and printing the formatted progress message.

You may want something like the following:

```
76% (152 of 200) [===============     ] 36s remaining
``` 

Which on windows with conemu will display as:

![Layer stack for the system](/img/bashprogress/bashProgress.png)
***Propagated Progress Indicators*** *The progress indicators are detected by ConEmu and displayed in the window title and on the taskbar in Windows*
{: class="figure"}

This can be achieved with some basic stats as shown below:

{% highlight shell %}
START_DATE=$(date +%s)
TOTAL=$(wc -l <Source list file> | cut "-d " -f1)
COUNT=0

for i in <source list>
do
    COUNT=$((COUNT+1))
    PERCENT=$((100 * COUNT / TOTAL))
    BARLEN=$((PERCENT / 5))
    TIME=$(( $(date +%s) - START_DATE ))
    REMAINING=$(( ((100*TIME/COUNT) * (TOTAL-COUNT))/100 ))

    printf "\r%d%% (%d of %d)  [" "$PERCENT" "$COUNT" "$TOTAL"
    for i in {1..20}
    do
        if [ "$i" -gt "$BARLEN" ]
        then printf "%s" " "
        else printf "%s" "="
        fi
    done

    printf "] %ds remaining       " "$REMAINING"

    #do the work
done
{% endhighlight %}

# Breakdown

`TOTAL`, `COUNT`, and `PERCENT` are pretty straight forward and self explanatory.

The `START_DATE` captures the Unix time when the process started, and `TIME` stores the time taken to process so far. `REMAINING` is a linear approximation of the remaining time given the time taken so far, and the number of items remaining.

`BARLEN` is a transformation of the percentage into a discrete range of 0 to 20.

The completion statistics are then printed, and the opening brace of the progress bar.

The progress bar is printed one character at a time, with a `=` for done and a ` ` for not yet done.

The closing brace is then printed and the remaining time estimate with space padding to cater for number size differences which would leave trailing characters when they get shorter.

# Packaged Up

The progress bar logic can be packaged up into bash functions making them easy to re-use as an initialise and increment function:

{% highlight shell %}
function init_progress() {
	START_DATE=$(date +%s)
	TOTAL="$1"
	COUNT=0
}

function inc_progress() {
	COUNT=$((COUNT+1))
	PERCENT=$((100 * COUNT / TOTAL))
	BARLEN=$((PERCENT / 5))
	TIME=$(( $(date +%s) - START_DATE ))
	REMAINING=$(( ((100*TIME/COUNT) * (TOTAL-COUNT))/100 ))

	printf "\r%d%% (%d of %d)  [" "$PERCENT" "$COUNT" "$TOTAL" 
	for i in {1..20}
	do
		if [ "$i" -gt "$BARLEN" ]
		then printf "%s" " "
		else printf "%s" "="
		fi
	done

	printf "] %ds remaining     " "$REMAINING"
}

init_progress "$(wc -l <Source list file> | cut "-d " -f1)"

for i in <Source list file>
do
    inc_progress

    #do the work
done
{% endhighlight %}

# Notes

Bash only supports integers, therefore your number ranges should be projected into a range such that this is not an issue (eg, for the remaining time estimate).

ConEmu will cleverly detect the precentage when it is surrounded by whitespace it seems.

# Conclusion

Bash logic for providing a progress indicator in your shell scripts has been presented. It features the following:

-  Count of total
-  Percentage completed
-  Progress bar
-  Estimated time remaining
-  Display of progress in ConEmu (windows)
-  Encapsulated in functions for easy re-use

