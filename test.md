## GitHub Setup — do this once
> We need a Git repo that the Config Server can read.

| Step | What to do | Expected Result |
|------|------------|-----------------|
| 1 | Sign in to <https://github.com> (or create an account). Grab your **username** from the top-right avatar; you’ll replace `<your-username>` below. | — |
| 2 | Click **➕ New → New repository**.<br>• **Name**: `config-repo`<br>• **☑ Add a README** ✔<br>Click **Create repository**. | Repo page shows *README.md* |
| 3 | Still on GitHub, click **Add file → Create new file**.<br>• **Filename**: `user-service.properties`<br>• Put only a comment for now: `# placeholder`<br>Scroll down → **Commit new file**. | File appears in the repo |
| 4 | In any terminal, clone locally (create the parent folder first if it doesn’t exist):<br>`mkdir -Force C:\SpringCloud\others`<br>`git clone https://github.com/<your-username>/config-repo.git C:\SpringCloud\others\config-repo` | *C:\SpringCloud\others\config-repo* contains README and `user-service.properties` |

*Skip Step 4 only if you already have both the GitHub repo **and** a correct local clone.*
