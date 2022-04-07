# Documentation setups

## Vagrant

[Trevor's vagrant box setup](https://github.com/tcooper/rocky-linux-wikibox)

## WSL

- Download the [Cloud image](https://github.com/rocky-linux/sig-cloud-instance-images/actions/workflows/build.yml)
- Get the actual disk out of the image (extract rocky-8.5-docker.tar)
- `wsl --import <machine-name> <path-to-vm-dir> <path-to/rocky-8.5-docker.tar>`
- `wsl -d <machine-name>`
- `dnf -y update`
- `dnf -y install git python39-pip`
- `python3.9 -m pip install -U pip`
- `git clone <path-to-git-project>`
- `cd <git-project-name>`
- Sometimes you will need to switch the branch here
- `python3.9 -m pip install -r requirements.txt`
- If you want to publish it somewhere else run `mkdocs build` and move the `site` content to the correct destination
- If you just want to look at the output run `mkdocs serve 2>&1 | tee ./mkdocs.serve.log`

With WSL you can also startup this WSL after the first setup in VS Code to directly change stuff in the content, while the last line is running, and it will update automatically in the browser!
