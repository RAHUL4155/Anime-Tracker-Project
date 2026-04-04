# Anime Tracker

Anime Tracker is a feature-rich, local-first web application built with [Vue.js 3](https://vuejs.org/) and Vite that helps you track your anime watching progress, discover new series, and manage episode reminders.

##  Features

*   **Personal Anime List**: Manage your anime collection with statuses (Watching, Completed, Plan to Watch, On Hold, Dropped).
*   **Search & Discovery**: Integrated with [Jikan API (MyAnimeList)](https://jikan.moe/) to search for any anime.
*   **Progress Tracking**: Track watched episodes and see visual progress bars.
*   **Detailed Statistics**: View insights into your watching habits, including total episodes, completion rates, and average ratings.
*   **Smart Reminders**: Set custom reminders for upcoming episodes with browser notifications.
*   **Advanced Filtering**: Sort and filter your list by genre, year, language, rating, and watch status.
*   **Data Management**: Fully local storage based. Export and Import your data easily via JSON to backup or sync across devices.
*   **Theming**: Built-in Dark Mode and Light Mode.
*   **Personal Notes**: Add private notes to any anime entry.

##  Tech Stack

*   **Framework**: Vue.js 3 (Composition API)
*   **Build Tool**: Vite
*   **Styling**: Custom CSS3 variables
*   **API**: Jikan API v4
*   **Persistence**: LocalStorage

##  Installation

1.  Clone the repository:
    ```bash
    git clone https://github.com/yourusername/anime-tracker.git
    cd anime-tracker
    ```

2.  Install dependencies:
    ```bash
    npm install
    ```

3.  Run the development server:
    ```bash
    npm run dev
    ```

4.  Open your browser and visit: `http://localhost:3000`

##  Usage

*   **Adding Anime**: Go to the 'Discover' tab, search for an anime, and click 'Add to My Anime'.
*   **Updating Progress**: Use the '+' and '-' buttons on cards in 'My Anime List' to update episode counts.
*   **Data Backup**: Click 'Export Data' in the header to get a JSON string of your entire library. Save this safely!

##  Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

##  License

This project is open source and available under the [MIT License](LICENSE).
