# Live Visual Sentiment Engine

## 1. Overview

This module is designed to provide a "Live Sentiment" score for financial assets. It functions by capturing screenshots of real-time financial charts from TradingView platform using selenium and using a multimodal Large Language Model (LLM) to perform visual sentiment analysis. The output is a sentiment score that can be integrated in the composite score of the scoring engine.
The system is built to be asynchronous, robust, and handle a batch of incoming prediction requests efficiently by only processing unique assets(right now its tested for 3 unique assets- BTC,NVDA,TSLA).

## 2. Architecture

The engine follows a orchestrated design. A central `SentimentEngine` coordinates the workflow between three specialized components: a `ScreenshotCapturer`, an `LLMSentimentExtractor`, and a `SentimentAggregator`. This design ensures that each part of the process is decoupled and can be maintained or upgraded independently.

## 3. File Structure

The module is organized as a Python package (`sentiment_analysis_scoring`) with a test script and configuration files at the top level.

# 4. Complete Structure
sentiment_analysis_scoring/

├── sentiment_engine/
│   ├── init.py

│   ├── engine.py

│   ├── llm_sentiment_extractor.py

│   ├── screenshot_capturer.py

│   ├── sentiment_aggregator.py

│   └── models.py

├── test.py

├── sentiment_config.yaml

└── requirements.txt

## 4. Component Breakdown

Each file in the directory has a specific and crucial role in the pipeline.

### `sentiment_engine/`
This directory is the core Python package containing all the application logic.

* #### `__init__.py`
    This file marks the `sentiment_engine` directory as a Python package, allowing its modules to be imported. It also defines the public API of the package by specifying which classes are exported.

* #### `engine.py`
    This is the central orchestrator. Its `SentimentEngine` class receives a batch of prediction messages, extracts the unique assets, and manages the end-to-end process of capturing screenshots and extracting sentiment for each. It also handles the global timeout and caching logic.

* #### `screenshot_capturer.py`
    This module handles all browser automation using Selenium 4. Its `ScreenshotCapturer` class creates a new, clean, headless browser instance for each unique asset, navigates to TradingView, and captures a full-page screenshot. This stateless, "new-driver-per-asset" design is critical for ensuring data integrity and preventing incorrect screenshots.

* #### `llm_sentiment_extractor.py`
    This is the brain of the operation. The `LLMSentimentExtractor` class is responsible for loading the specified multimodal LLM and processor from Hugging Face(that we have downloaded on this VM). It takes a screenshot, constructs a detailed prompt, and passes both to the model to get the raw `visual`, `momentum`, and `narrative` sentiment scores.

* #### `sentiment_aggregator.py`
    This module refines the raw AI output. The `SentimentAggregator` class takes the raw scores from the LLM and calculates a single, stable sentiment score. It does this by applying configured weights, temporal smoothing (Exponential Moving Average), and outlier detection.

* #### `models.py`
    This file defines the data structures that are passed between the different components. It uses Python's `dataclasses` to ensure consistency for objects like `Screenshot` and `SentimentData`.

### Top-Level Files

* #### `test.py`
    This is the main entry point for running a test of the sentiment engine. It simulates the real-world environment by generating a batch of synthetic prediction messages (as if they were coming from a Kafka stream) and passes them to the engine. It then prints the final results and health status.

* #### `sentiment_config.yaml`
    This is the central configuration file. It allows you to control all critical parameters without changing the Python code, including the LLM model ID, timeouts, caching TTL, and the weights used for sentiment aggregation.

* #### `requirements.txt`
    This file lists all the necessary Python libraries and their versions required to run the project.

---

## 5. How It All Comes Together: The Workflow

The system executes in a clear, sequential flow managed by the `SentimentEngine`.

1.  **Execution Starts**: A user or an upstream service runs `python test.py`.
2.  **Simulate Input**: The `test.py` script generates a batch of prediction data (e.g., 20 predictions across 3 unique assets).
3.  **Engine Call**: The batch is passed to the `SentimentEngine.score()` method.
4.  **Deduplication**: The engine intelligently extracts the list of **unique assets** from the batch. This is a key efficiency step.
5.  **Screenshot Capture**: For each unique asset, the `ScreenshotCapturer` is called. It launches a fresh, headless browser, navigates to the correct TradingView URL, waits for the page to load, resizes the window to the full page height, and captures a complete screenshot. The browser is then immediately closed.
6.  **AI Analysis**: The captured screenshot and a carefully engineered prompt are passed to the `LLMSentimentExtractor`. The extractor feeds this data to the loaded multimodal LLM, which returns a JSON object with raw sentiment scores.
7.  **Score Aggregation**: The raw scores are passed to the `SentimentAggregator`, which calculates the final, smoothed score for that asset.
8.  **Return Results**: Once all unique assets have been scored, the `SentimentEngine` returns a final dictionary mapping each unique asset to its sentiment score (e.g., `{'NASDAQ:TSLA': 0.65, 'COINBASE:BTC-USD': 0.45}`).
9.  **Display Output**: The `test.py` script receives this dictionary and prints the final, human-readable results to the terminal.

If the entire process exceeds the global timeout (e.g., 120 seconds), the engine gracefully stops and returns a dictionary of neutral scores (0.5) for all assets.

### Important Note:
 1-The sentiment scores are not yet being used in the composite score calculation formula, it exists as a seperate layer as of now.
 #Also this module has been tested on sythetic data( ie the assets BTC,NVDA and TSLA) these specific assets used for testing are specified in the test.py file, and not on any kind of data being recieved from any kafka stream or some other source.

 2-If you would like to test this module for yourself, please come to this path - ~/cq-scoring/src/cq_scoring_engine/sentiment_analysis_scoring$ and run python test.py, you will see how the module is capturing and scoring. Also you can see the screenshots as well, as they will be available in the same folder, and you can also check the accuracy of the image by checking the time in the image and the current time, there will not be any significant difference between these two.