# Document-Image-Extraction-repo
This repo offers a single-file Streamlit app (app.py) that extracts images from PDFs, DOCX, PPTX, and HTML. It can optionally use Google Gemini (GenAI) to generate natural-language descriptions of figures, charts, and diagrams by analyzing surrounding context, combining parsing with AI insight.

# Overview

This Streamlit application provides an end-to-end workflow for document image analysis. Key capabilities include:

1. Uploading local files or processing remote URLs for supported formats (PDF, DOCX, PPTX, HTML).
2. Extracting all embedded images while automatically filtering out small or low-quality images via a dynamic min_size threshold computed from the document’s own contents.
3. Optionally generating semantic descriptions for each extracted image using Google Gemini (GenAI), leveraging the surrounding document text for contextual accuracy.
4. Displaying extracted images side by side with AI-generated descriptions in the web interface.
5. Downloading all results, including images and a text file of descriptions, packaged neatly as a ZIP archive.

This makes the app useful for research papers, business reports, educational material, and online content where figures and diagrams are central to understanding.

# Usage

1. Clone the repository and set up a Python environment:

python -m venv venv
source venv/bin/activate  # On Windows use: venv\Scripts\activate
pip install -r requirements.txt

2. Set your GOOGLE_API_KEY in the environment or a .env file if you want Gemini-based descriptions. Without it, you can still run extractions by enabling Dry run in the UI to skip AI calls.

3. Run the Streamlit app:

streamlit run app.py

4. Open the Streamlit interface, upload a file or paste a URL, choose an extraction mode from the sidebar, and click Run Extraction. Results will be displayed interactively and can be downloaded.

# Notes

1. The project is designed as a single-file pipeline for portability and quick deployment. No external microservices or complex dependencies are required beyond the listed packages.
2. The PDF extractor leverages PyMuPDF, DOCX and PPTX are parsed directly from their ZIP containers, and HTML is processed with BeautifulSoup.
3. The adaptive threshold ensures that icons or decorative elements are skipped, focusing instead on meaningful figures.
4. Gemini integration is optional but provides powerful contextual analysis of charts, diagrams, or labeled images.
5. The app supports both local and cloud use cases, from personal document analysis to research automation.
