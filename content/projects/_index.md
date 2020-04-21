+++
title = "Projects"
description = "Pat Gaffney's projects"
sort_by = "date"
template = "projects.html"
+++

# 👨‍💻 Active Projects

## dotfiles

My personal [collection of dotfiles][dotfiles]. They are an evolving collection of random hacks and aliases. I wrote [an article][bash-article] once about my `bash` setup.

## Sublime Text Linters

I have become a [Sublime Text][st3] stan, and in my standom, I've come to write and maintain a few different [SublimeLinter][sub-linter] plugins:

- [SublimeLinter-contrib-terraform][sl-tf-github] wraps `terraform validate`. You can get it from [Package Control][sl-tf-pkgctl].
- [SublimeLinter-contrib-staticheck][sl-sc-github] wraps `staticcheck`. You can get it from [Package Control][sl-sc-pkgctl].

# 📈 Recent & Ongoing Contributions

## Terraform.tmLanguage

I added the [`.sublime-syntax`][sublime-syntax] definition and updated to HCL2 in [early 2020][tf-pr] — subsequently became a contributor. Blame me for the broken [GitHub syntax][linguist], I guess. The package is hosted on [Package Control][tf-pkgctl].

# 💀 Mostly Dead Artifacts

## patdown

A [mostly defunct attempt][patdown] at writing a [CommonMark][commonmark]-compliant parser in C99 with no regular expressions and no external dependencies. I say _mostly defunct_ because I try not to write C anymore (for my health), but never say never.

## nba.py

A [fun little weekend experiment][nba-py] from the college days. It's a small CLI wrapper around the nba.com API to pretty print box scores. Was trying to get into Python at the time, which never really took.


[bash-article]: @/blog/two-shells-one-prompt.md
[commonmark]: https://commonmark.org/
[dotfiles]: https://github.com/patrickrgaffney/dotfiles
[linguist]: https://github.com/github/linguist
[nba-py]: https://github.com/patrickrgaffney/NBA-CLI
[patdown]: https://github.com/patrickrgaffney/patdown
[sl-sc-github]: https://github.com/patrickrgaffney/SublimeLinter-contrib-staticcheck
[sl-sc-pkgctl]: https://packagecontrol.io/packages/SublimeLinter-contrib-staticcheck
[sl-tf-github]: https://github.com/patrickrgaffney/SublimeLinter-contrib-terraform
[sl-tf-pkgctl]: https://packagecontrol.io/packages/SublimeLinter-contrib-terraform
[st3]: https://sublimetext.com/
[sub-linter]: http://www.sublimelinter.com/en/stable/
[sublime-syntax]: https://www.sublimetext.com/docs/3/syntax.html
[tf-pkgctl]: https://packagecontrol.io/packages/Terraform
[tf-pr]: https://github.com/alexlouden/Terraform.tmLanguage/pull/39