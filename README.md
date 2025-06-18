# sentinel2-wetland-classification-timeseries
Python pipeline for processing multi-temporal Sentinel-2 satellite imagery for wetland classification. Fetches cloud-filtered data across multiple years, applies cloud masking, and computes median composite images. Imagery is used to train a regression model and apply this over multiple years. Built with stackstac, pystac_client, dask and scikit.
