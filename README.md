# Personal blog using Hugo

To write a new post
1. Run `hugo server` and open localhost:1313
2. Write post in content/posts/
3. Live reload on save!
    - Since I'm using WSL2, ensure project dir is [in WSL's filesystem](https://discourse.gohugo.io/t/a-cautionary-tale-mixing-windows-and-wsl-windows-subsystem-for-linux/17896) and not the mounted windows' drive for the live reload to work. This is cos windows doesn't support the inotify events from WSL2. There is in fact an [open thread](https://github.com/microsoft/WSL/issues/4739) since 2019 about this issue. Same issue I faced when using Jekyll live reload as well which was [reported since 2016](https://github.com/microsoft/WSL/issues/216) and worked around by polling.

In case I need to make changes in themes/PaperMod
1. cd into themes/PaperMod, commit the change
2. cd back to project dir (parent repo), commit themes/PaperMod to point to the new commit
3. Push the submodule repo first before pushing the project repo