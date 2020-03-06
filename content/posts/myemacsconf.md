+++
title = "我的Emacs配置"
author = ["keke"]
date = 2020-03-06T00:00:00+08:00
draft = false
creator = "Emacs 26.3 (Org mode 9.3.6 + ox-hugo)"
+++

## 初始化 straight.el 插件管理 {#初始化-straight-dot-el-插件管理}

```emacs-lisp
(defvar bootstrap-version)
(let ((bootstrap-file
       (expand-file-name "straight/repos/straight.el/bootstrap.el" user-emacs-directory))
      (bootstrap-version 5))
  (unless (file-exists-p bootstrap-file)
    (with-current-buffer
	(url-retrieve-synchronously
	 "https://raw.githubusercontent.com/raxod502/straight.el/develop/install.el"
	 'silent 'inhibit-cookies)
      (goto-char (point-max))
      (eval-print-last-sexp)))
  (load bootstrap-file nil 'nomessage))
```


## 预下载基础包 {#预下载基础包}

```emacs-lisp
(straight-use-package 'use-package)
```


## 主程序的一些设置 {#主程序的一些设置}


### UI {#ui}

```emacs-lisp
 (menu-bar-mode -1)
 (tool-bar-mode -1)
 (scroll-bar-mode -1)
 (global-linum-mode 1)
;; (setq inhibit-startup-message t)
 (global-hl-line-mode t)
```


### 透明度 {#透明度}

```emacs-lisp
;;(add-to-list 'default-frame-alist'(ns-transparent-titlebar . t))
;;(add-to-list 'default-frame-alist'(ns-appearance . dark))
;;(add-to-list 'default-frame-alist'(alpha . (80 . 75)))
```


### 默认功能设置 {#默认功能设置}

```emacs-lisp
;; 自动加载外部修改过的文件
(global-auto-revert-mode 1)
;; 关闭自己生产的保存文件
(setq auto-save-default nil)
;; 关闭自己生产的备份文件
(setq make-backup-files nil)
;; 选中某个区域继续编辑可以替换掉该区域
(delete-selection-mode 1)
;; 设置h 文件默认为c++文件
(add-to-list 'auto-mode-alist '("\\.h\\'" . c++-mode))
```


### 主题&字体 {#主题-and-字体}

```emacs-lisp
;; 设置主题
;;(add-to-list 'custom-theme-load-path "~/.emacs.d/themes")
;;(load-theme 'dracula t)
;; 设置字体
;;(set-frame-(format "message" format-args)ont "Sarasa Mono SC 14")
```


#### doom-modeline {#doom-modeline}

```emacs-lisp
(straight-use-package 'doom-modeline)
(use-package doom-modeline
  :init (doom-modeline-mode 1))
```


#### doom-themes {#doom-themes}

```emacs-lisp
(straight-use-package 'doom-themes)
(use-package doom-themes
  :config
  ;; Global settings (defaults)
  (setq doom-themes-enable-bold t    ; if nil, bold is universally disabled
	doom-themes-enable-italic t) ; if nil, italics is universally disabled
  (load-theme 'doom-tomorrow-day t)

  ;; Enable flashing mode-line on errors
  (doom-themes-visual-bell-config)

  ;; Enable custom neotree theme (all-the-icons must be installed!)
  (doom-themes-neotree-config)
  ;; or for treemacs users
  (setq doom-themes-treemacs-theme "doom-colors") ; use the colorful treemacs theme
  (doom-themes-treemacs-config)

  ;; Corrects (and improves) org-mode's native fontification.
  (doom-themes-org-config))
```


## 包的配置 {#包的配置}


### ox-hugo {#ox-hugo}

```emacs-lisp
(straight-use-package 'ox-hugo)
(use-package ox-hugo
  :after ox)
```


### yasnippet {#yasnippet}

```emacs-lisp
(straight-use-package 'yasnippet)
(straight-use-package 'yasnippet-snippets)
(use-package yasnippet
  :commands
  (yas-reload-all)
  :init
  (add-hook 'prog-mode-hook #'yas-minor-mode))
```


### IVY all {#ivy-all}

```emacs-lisp
(straight-use-package 'ivy)
(straight-use-package 'counsel)
(straight-use-package 'swiper)
(straight-use-package 'all-the-icons-ivy-rich)
(use-package ivy
  :init
  (ivy-mode 1)
  (setq ivy-use-virtual-buffers t)
  (setq enable-recursive-minibuffers t))
(use-package all-the-icons-ivy-rich
  :init (all-the-icons-ivy-rich-mode 1))
(use-package ivy-rich
  :init (ivy-rich-mode 1))
```


## 键位配置 {#键位配置}

```emacs-lisp
(global-set-key (kbd "C-c p") 'keke-run-current-file)
;;IVY
(global-set-key "\C-s" 'swiper)
(global-set-key (kbd "C-c C-r") 'ivy-resume)
(global-set-key (kbd "<f6>") 'ivy-resume)
(global-set-key (kbd "M-x") 'counsel-M-x)
(global-set-key (kbd "C-x C-f") 'counsel-find-file)
(global-set-key (kbd "<f1> f") 'counsel-describe-function)
(global-set-key (kbd "<f1> v") 'counsel-describe-variable)
(global-set-key (kbd "<f1> l") 'counsel-find-library)
(global-set-key (kbd "<f2> i") 'counsel-info-lookup-symbol)
(global-set-key (kbd "<f2> u") 'counsel-unicode-char)
(global-set-key (kbd "C-c g") 'counsel-git)
(global-set-key (kbd "C-c j") 'counsel-git-grep)
(global-set-key (kbd "C-c k") 'counsel-ag)
(global-set-key (kbd "C-x l") 'counsel-locate)
(global-set-key (kbd "C-S-o") 'counsel-rhythmbox)
(define-key minibuffer-local-map (kbd "C-r") 'counsel-minibuffer-history)
```


## keke-run-current-file(fork to leexah) {#keke-run-current-file--fork-to-leexah}

```emacs-lisp
(defvar keke-run-current-file-before-hook nil "Hook for `keke-run-current-file'. Before the file is run.")

(defvar keke-run-current-file-after-hook nil "Hook for `keke-run-current-file'. After the file is run.")

(defun keke-run-current-file ()
  (interactive)
  (let (
	($outputb "*keke-run output*")
	(resize-mini-windows nil)
	($suffix-map
	 `(
	   ("ts" . "node")
	   ("html" . "firefox-bin")
	   ))
	   $fname
	   $fSuffix
	   $prog-name
	   $cmd-str)
	 (when (not (buffer-file-name)) (save-buffer))
	 (when (buffer-modified-p) (save-buffer))
	 (setq $fname (buffer-file-name))
	 (setq $fSuffix (file-name-extension $fname))
	 (setq $prog-name (cdr (assoc $fSuffix $suffix-map)))
	 (setq $cmd-str (concat $prog-name " \""   $fname "\" &"))
	 (run-hooks 'keke-run-current-file-before-hook)
	 (if $prog-name
	     (progn
	       (message "Running")
	       (shell-command $cmd-str $outputb ))
	   (error "No recognized program file suffix for this file."))))
(run-hooks 'keke-run-current-file-after-hook)
```