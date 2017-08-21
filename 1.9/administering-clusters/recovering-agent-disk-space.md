---
post_title: Recovering Agent Disk Space
menu_order: 900
---

If tasks fill up the reserved volume of an agent node, there are a few options to recover space:

- If the work directory is on a separate volume (as recommended in [Agent nodes](/docs/1.10/installing/custom/system-requirements/#agent-nodes), then you can empty this volume and restart the node.

- If the work directory is not on a separate volume, it may be necessary to check each component's healthiness and restart them. 

If neither of these approaches work, you may need to re-image the node. 
