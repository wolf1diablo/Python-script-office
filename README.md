# Python-script-office data scraper
Add script I currently find that organizations need 







    import requests
    from bs4 import BeautifulSoup
    import pandas as pd
    import time
    import random

    print("working RN")

    def scrape_player_data(player_urls):
        """
        Scrapes detailed player information from various sources for a list of players,
        handling potential errors and rate limiting.
        """

        player_data = []
        headers = {
            'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36'
        }

        for player_name, urls in player_urls.items():
            player_info = {"Name": player_name}

            if "transfermarkt" in urls:
                try:
                    # Construct the URL for the player's transfers page on Transfermarkt.
                    transfer_url = urls["transfermarkt"].replace("/profil/", "/transfers/")
                    response = requests.get(transfer_url, headers=headers)
                    response.raise_for_status()
                    soup = BeautifulSoup(response.content, "html.parser")
                    bsoup = BeautifulSoup(requests.get(urls["transfermarkt"], headers=headers).content, "html.parser")

                    # Extract demographic data from the player's main profile page, not transfers page.
                    try:
                        main_page_soup = BeautifulSoup(requests.get(urls["transfermarkt"], headers=headers).content, "html.parser")
                        age_text = main_page_soup.find("span", {"itemprop": "birthDate"}).text
                        player_info["Age"] = age_text.split("(")[1].split(")")[0]
                    except (AttributeError, IndexError, requests.exceptions.RequestException):
                        player_info["Age"] = None

                    try:
                        main_page_soup = BeautifulSoup(requests.get(urls["transfermarkt"], headers=headers).content, "html.parser")
                        player_info["Nationality"] = main_page_soup.find("span", {"itemprop": "nationality"}).text.strip()
                    except (AttributeError, requests.exceptions.RequestException):
                        player_info["Nationality"] = None

                    try:
                        main_page_soup = BeautifulSoup(requests.get(urls["transfermarkt"], headers=headers).content, "html.parser")
                        height_element = main_page_soup.find("span", {"itemprop": "height"})
                        player_info["Height"] = height_element.text.strip() if height_element else None
                    except (AttributeError, requests.exceptions.RequestException):
                        player_info["Height"] = None

                    try:
                        main_page_soup = BeautifulSoup(requests.get(urls["transfermarkt"], headers=headers).content, "html.parser")
                        weight_element = main_page_soup.find("span", {"itemprop": "weight"})
                        player_info["Weight"] = weight_element.text.strip() if weight_element else None
                    except (AttributeError, requests.exceptions.RequestException):
                        player_info["Weight"] = None

                    career_history = []
                    # Find the table containing career history from the transfers page.
                    transfer_history_box = bsoup.find ( "div", {"class" : "grid__cell grid__cell--center tm-player-transfer-history-grid__old-club"} )
                    if  transfer_history_box :
                        club_link = transfer_history_box.find_all ( "a", {"class" : "tm-player-transfer-history-grid__club-link"} )
                        for club_name in club_link :
                            club_name = club_link.text.strip ()
                            career_history.append ( club_name )

                    player_info["Previous Clubs"] = ", ".join(career_history)

                    transfer_fees = []
                    transfer_fee_elements = bsoup.find_all("td", {"class": "rechts hauptlink"})
                    for fee_element in transfer_fee_elements:
                        transfer_fees.append(fee_element.text.strip())
                    player_info["Transfer Fees"] = ', '.join(transfer_fees)

                except requests.exceptions.RequestException as e:
                    print(f"Error scraping Transfermarkt for {player_name}: {e}")
                except Exception as e:
                    print(f"An unexpected error occurred while scraping Transfermarkt for {player_name}: {e}")
                time.sleep(random.uniform(1, 3))

            if "sofascore" in urls:
                try:
                    response = requests.get(urls["sofascore"], headers=headers)
                    response.raise_for_status()
                    soup = BeautifulSoup(response.content, "html.parser")

                    try:
                        goals_element = soup.select_one('div[data-testid="player-goals"]')
                        player_info["Goals"] = goals_element.text.strip() if goals_element else None
                    except AttributeError:
                        player_info["Goals"] = None

                    try:
                        assists_element = soup.select_one('div[data-testid="player-assists"]')
                        player_info["Assists"] = assists_element.text.strip() if assists_element else None
                    except AttributeError:
                        player_info["Assists"] = None

                    try:
                        minutes_played_element = soup.select_one('div[data-testid="player-minutesPlayed"]')
                        player_info["Minutes Played"] = minutes_played_element.text.strip() if minutes_played_element else None
                    except AttributeError:
                        player_info["Minutes Played"] = None

                    try:
                        passes_element = soup.select_one('div[data-testid="player-accuratePasses"]')
                        player_info["Passes"] = passes_element.text.strip() if passes_element else None
                    except AttributeError:
                        player_info["Passes"] = None

                except requests.exceptions.RequestException as e:
                    print(f"Error scraping SofaScore for {player_name}: {e}")
                except Exception as e:
                    print(f"An unexpected error occurred while scraping SofaScore for {player_name}: {e}")
                time.sleep(random.uniform(1, 3))

            if "flashscore" in urls:
                try:
                    response = requests.get(urls["flashscore"], headers=headers)
                    response.raise_for_status()
                    soup = BeautifulSoup(response.content, "html.parser")

                    ratings = []
                    rating_elements = soup.find_all('span', class_='rating')
                    for rating_element in rating_elements:
                        ratings.append(rating_element.text.strip())
                    player_info["FlashScore Ratings"] = ', '.join(ratings) if ratings else "Ratings not available"
                except requests.exceptions.RequestException as e:
                    print(f"Error scraping FlashScore for {player_name}: {e}")
                except Exception as e:
                    print(f"An unexpected error occurred while scraping FlashScore for {player_name}: {e}")
                time.sleep(random.uniform(1, 3))

            if "fbref" in urls:
                try:
                    response = requests.get(urls["fbref"], headers=headers)
                    response.raise_for_status()
                    soup = BeautifulSoup(response.content, "html.parser")

                    try:
                        passes_completed_element = soup.find('div', {'data-stat': 'passes_completed'}).find('strong')
                        player_info["FBref Passes Completed"] = passes_completed_element.text if passes_completed_element else None
                    except AttributeError:
                        player_info["FBref Passes Completed"] = None

                    try:
                        shots_total_element = soup.find('div', {'data-stat': 'shots_total'}).find('strong')
                        player_info["FBref Shots Total"] = shots_total_element.text if shots_total_element else None
                    except AttributeError:
                        player_info["FBref Shots Total"] = None
    
                    try:
                        tackles_element = soup.find('div', {'data-stat': 'tackles'}).find('strong')
                        player_info["FBref Tackles"] = tackles_element.text if tackles_element else None
                    except AttributeError:
                        player_info["FBref Tackles"] = None

                except requests.exceptions.RequestException as e:
                    print(f"Error scraping FBref for {player_name}: {e}")
                except Exception as e:
                    print(f"An unexpected error occurred while scraping FBref for {player_name}: {e}")
                time.sleep(random.uniform(1, 3))

            player_data.append(player_info)

        df = pd.DataFrame(player_data)
        pd.set_option('display.max_columns', None)
        return df

    player_urls = {
        "Lionel Messi": {
            "transfermarkt": "https://www.transfermarkt.com/lionel-messi/profil/spieler/28003",
            "sofascore": "https://www.sofascore.com/player/lionel-messi/14736",
            "flashscore": "https://www.flashscore.com/player/messi-lionel/G0M7h9g7/",
            "fbref": "https://fbref.com/en/players/d70ce98e/Lionel-Messi"
        },
    }

        df_player_data = scrape_player_data(player_urls)
        print(df_player_data)
