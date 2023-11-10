+++
title = "Documenting unscheduled tasks in Org mode"
date = 2023-11-09T22:58:00+01:00
tags = ["org", "emacs"]
draft = false
+++

A common occurrence in my daily work is to provide on-the-spot support to colleagues and/or users. These can occur in a multitude of different ways: office or mobile phone, calls via a virtual meeting tool, in person by showing up in my office, while walking to a lab, etc. Most importantly, they are unscheduled, usually short (less than 15min), and inconsequential.

I decided to develop a function to quickly document these interactions in my Org workflow. I plan to write a post (or a series) on how I use Org mode in my daily work, but suffice to say for now that my tasks, planned and ongoing, are all available in a single org file, `todos.org`. The main headings reflect what I consider to be my main responsibilities, and the corresponding subheadings describe individual tasks.

All these support tasks will be hosted under the heading `Quick Support`. I wanted something quick, so my function asks for how much time was spent, a heading title, and if I will continue to work on my last task. The first value is easy to know, but one can also estimate; the main purpose is to clock out of the current task I was working on at the correct time. The heading title is just a brief description to help me flesh out, if required, the entry later on. And since sometimes providing support really needs to be compensated with a coffee and a walk, the function asks if I will continue work or not.

Below is the function code and my keyboard shortcut:

```emacs-lisp
;; Function
(defun bjf/quick-support (minutes-started)
  ;; Function to track the time spent with on the spot support, i.e. unscheduled dialogues/actiosn with colleagues. It requests how many minutes I worked on it, a title and if I should continue clokcing the last task.
  (interactive "NMinutes:")
  (save-excursion
	(let ((support-start-time
		(encode-time (decoded-time-add (decode-time) (make-decoded-time :minute (- minutes-started))))))
	  (with-current-buffer "todos.org"
		  ;; If clocked in a task, stopped it when support started
		  (org-clock-out nil t support-start-time)
		  (goto-char (org-find-exact-headline-in-buffer "Quick Support" "todos.org"))
		  (org-goto-sibling)
		  (org-insert-subheading t)
		  (insert (concat (read-string "Title: ") "\t\n:CLOCKING:\n CLOCK: "
			  (format-time-string "[%Y-%m-%d %a %H:%M]" support-start-time) "--"
			  (format-time-string "[%Y-%m-%d %a %H:%M]") " => "
			  (format "%02d:%02d" (/ minutes-started 60) (% minutes-started 60))
			  "\n:END:\n"))))
	  (if (y-or-n-p "Continue last task?") (org-clock-in-last))))

(global-set-key (kbd "C-c q") #'bjf/quick-support)
```

At the end of the day, I usually review these entries and assign tags, which I think reflect what was lacking from my side for the support to take place for the first time. If a task needs follow-up, I just treat it like any other one on that file (logbook, TODO state, etc.); otherwise, they are all considered close.

It is surprising how much these little pebbles of time can accumulate over the course of a month. My monthly clock report stated that 15% of my time was being spent on these tasks when I first started using this function. I use this information to create new team tasks based on the more prominent tags, which are usually something related to _Documentation_ and _Workflows_. Colleagues and users definitely come much better nowadays, allowing us to not only quickly get to the core of the issue or question but also dive into alternate solutions.
