# Forward-Search-Installation-Guide
This contains step-by-step guide to set up Forward Search for a particular domain. Input is a URL template for the target domain and output is a web-based entity-semantic search engine for the domain.

# Crawl Data
You need to install Jupyter Notebook to open file `PrepareUrlList.ipynb`. Change the domain in cell 6, e.g., `domain = "forwarddatalab.org"`, and run all cells. If necessary, please install missing packages by using pip, such as `pip install url_normalize`. Then you will find a new CSV file containing all urls corresponding to the domain in the same folder as the notebook file.
