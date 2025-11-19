# Weights and Biases
Is AI developer platform to train and fine-tune models, you can see while running loss plot, logs file and gpu status. 
* Sign up: https://wandb.ai/site/
* Enter your KTH email to get more space and projects
* Inside your script of a NN you will write the following:
```bash
# All your imports

import wandb
from datetime import datetime

# Once all the imports are specified
wandb.login()
wand.init(project="projectname", name"name_each_run"+ datetime.now().strftime("%Y-%m-%d_%H:%M:%S")
        #entity="aurorap-kth-royal-institute-of-technology-org"
    )

# If you have a config
wandb.config.update(config)

# Here you insert all your code, model, train, test
wand.log({'train loss': train_loss}, step=epoch) # or other things you want to save

# End of script
wandb.finish()
```

You can also create a sweep to perform hyperparameter search, follow guide: https://docs.wandb.ai/models/tutorials/sweeps 
