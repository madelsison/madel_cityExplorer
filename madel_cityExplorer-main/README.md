# City Explorer

Capstone Requirement for Web Development FRONT-END  NC III
## 📋 Prerequisites

1.  **Node.js** (Version 18 or higher recommended)
    * [Download Node.js here](https://nodejs.org/)
    * To check if installed, type `node -v` in your terminal.

2.  **Git**
    * [Download Git here](https://git-scm.com/)
    * To check if installed, type `git --version` in your terminal.

## 🚀 Installation & Local Development

Follow these steps to run the app on your local machine:

### 1. Clone the Repository

Open your terminal or command prompt and run:

```bash
git clone [https://github.com/gestaacinc/madel_cityExplorer.git](https://github.com/gestaacinc/madel_cityExplorer.git)
cd madel_cityExplorer

### 2\. Install Dependencies

This command installs all the necessary libraries (Vue, Tailwind, Vite, etc.) listed in `package.json`.

```bash
npm install
```

### 3\. Run the Development Server

Start the app in development mode with hot-reloading (changes update instantly):

```bash
npm run dev
```

> **Output:** You will see a local URL (usually `http://localhost:5173`). Open this link in your browser to view the app.

-----

## 🌐 Live Demo

You can view the live application here:
**[https://gestaacinc.github.io/madel\_cityExplorer/](https://www.google.com/search?q=https://gestaacinc.github.io/madel_cityExplorer/)**

-----

## 🛠️ Project Structure

  * **`src/App.vue`**: The main application logic, template, and styles.
  * **`src/main.js`**: The entry point that initializes Vue.
  * **`vite.config.js`**: Configuration for the build tool and base URL path.
  * **`tailwind.config.js`**: Configuration for the Tailwind CSS framework.

## 📚 API Reference

1.  **Open-Meteo API**:
      * *Purpose*: Provides Geocoding (Search) and Live Weather data.
      * *Status*: Primary, highly reliable.
2.  **Teleport API**:
      * *Purpose*: Provides "Quality of Life" scores and curated urban images.
      * *Status*: Secondary. If down, the app switches to "Basic Mode".
3.  **Wikipedia API**:
      * *Purpose*: Provides city images and "Popular Places" lists.
      * *Status*: Backup. Used automatically if Teleport is unavailable.
4.  **CorsProxy / AllOrigins**:
      * *Purpose*: Middleware used to bypass browser CORS security restrictions for external APIs.

 

*Developed by: Madel Sison*

 
