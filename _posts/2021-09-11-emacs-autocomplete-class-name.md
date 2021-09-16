---
layout: posts
title:  "Autocomplete Class name in Emacs"
excerpt: Generate the UpperCamelCase version of file_name in 1 line of elisp. Combined with yasnippet, we can autocomplete class name of ruby(or any other lang) file.
date:   2021-09-11 18:35:47 +0530
permalink: /blog/:year/:month/:day/:title.html
redirect_from:
  - /emacs-autocomplete-class-name
  - /emacs-autocomplete-class-name/
layout: single
author_profile: true
comments: true
tags: [emacs, elisp, yasnippet, ruby]
---
We will write some code to generate the classname from filepath in 1 line of elisp. Then we will use it with yasnippet to autocomplete classname.

Let us try to write the elisp function.

First, we will need to grab hold of the filename.
> `buffer-file-name` is a permanent local variable which contains the name of the file being visited in the current buffer, or nil if it is not visiting a file.

For example, visiting this file `users_controller.rb` in emacs and trying to get the `buffer-file-name` variable will return the absolute filepath.

![buffer-file-name](/assets/images/buffer-file-name.gif)


Once we have the filename, all we need to do is some simple text-processing and generate the class name.

1. Get the base path.
2. Convert the string to the upper camel case.

![generate-class-name](/assets/images/generate-class-name.gif)

Now we have an elisp function to get the class name from the file.

``` elisp
(defun camelCaseFileName()
  (s-upper-camel-case (file-name-base buffer-file-name)))
```

Using this function we can create a snippet and autocomplete class name.

![autocomplete-class](/assets/images/autocomplete-class.gif)


Full source code
``` elisp
(defun camelCaseFileName()
  "Get the camel case version of filename. Usefull when generating class name of ruby(or any other lang) file. You can use it with yasnippet."
  ;; # -*- mode: snippet -*-
  ;; # name: cls
  ;; # key: cls
  ;; # --
  ;; class `(camelCaseFileName)`
  ;;   $0
  ;; end
  (s-upper-camel-case (file-name-base buffer-file-name)))
```
