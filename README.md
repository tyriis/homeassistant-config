<!-- markdownlint-disable MD033 -->
<!-- markdownlint-disable MD041 -->

<!-- PROJECT SHIELDS -->
<!--
*** I'm using markdown "reference style" links for readability.
*** Reference links are enclosed in brackets [ ] instead of parentheses ( ).
*** See the bottom of this document for the declaration of the reference variables
*** for contributors-url, forks-url, etc. This is an optional, concise syntax you may use.
*** https://www.markdownguide.org/basic-syntax/#reference-style-links
-->

[![pre-commit][pre-commit-shield]][pre-commit-url]
[![taskfile][taskfile-shield]][taskfile-url]

<!-- PROJECT LOGO -->
<br />
<div align="center">
  <a href="https://gitlab.com/techtales/home-infra/gitops-home-assistant">
    <img src="docu/logo.png" alt="Logo" width="250" height="100%">
  </a>

  <h3 align="center">gitops-home-assistant</h3>

  <p align="center">
    GitOps workflows for home-assistant config files!
    <br />
    <br />
  </p>
</div>

<details>
  <summary style="font-size:1.2em;">Table of Contents</summary>
<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [About The Project](#about-the-project)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->
</details>

## About The Project

![Product Overview][product-overview]

Enable GitOps workflows for home-assistant configuration files.

Here's why:

- GitOps is my method of choice when applying configurations to a system in this case our system is home-assistant.
- Changes in the repo are auto applyed, it is transparent who has changed what and when.
- Just make your life easyer and edit your home-assistant config in your editor of choice and use known development workflows and tools :smile:

This repo was heavily inspired by [budimanjojo][budimanjojo-blog], thanks a lot for your work and the hints.

<p align="right">(<a href="#top">back to top</a>)</p>

<!-- MARKDOWN LINKS & IMAGES -->
<!-- https://www.markdownguide.org/basic-syntax/#reference-style-links -->

[product-overview]: docu/gitops-home-assistant-white.png
[budimanjojo-blog]: https://budimanjojo.com/2021/11/04/gitops-home-assistant-configurations/

<!-- Badges -->

[pre-commit-shield]: https://img.shields.io/badge/pre--commit-enabled-brightgreen?logo=pre-commit
[pre-commit-url]: https://github.com/pre-commit/pre-commit
[taskfile-url]: https://taskfile.dev/
[taskfile-shield]: https://img.shields.io/badge/Taskfile-Enabled-brightgreen?logo=task
