import random
from collections import deque
import time

class UnoCard:
    def __init__(self, color, value):
        self.color = color  # 'red', 'yellow', 'green', 'blue', 'wild'
        self.value = value  # number (0-9), 'skip', 'reverse', 'draw2', 'wild', 'wild_draw4'

    def __str__(self):
        if self.color == 'wild':
            if self.value == 'wild':
                return "Wild"
            else:
                return "Wild Draw 4"
        return f"{self.color.capitalize()} {self.value}"

    def __repr__(self):
        return self.__str__()

    def matches(self, other_card):
        """Check if this card can be played on top of another card"""
        if self.color == 'wild':
            return True
        if other_card.color == 'wild':
            return self.color == other_card.chosen_color
        return self.color == other_card.color or self.value == other_card.value

class UnoPlayer:
    def __init__(self, name):
        self.name = name
        self.hand = []

    def draw(self, card):
        self.hand.append(card)

    def play_card(self, card_index, current_card, game):
        if 0 <= card_index < len(self.hand):
            card = self.hand[card_index]
            if card.matches(current_card):
                # Handle special cards
                if card.value == 'skip':
                    game.skip_turn()
                elif card.value == 'reverse':
                    game.reverse_direction()
                elif card.value == 'draw2':
                    game.draw_cards(2)
                elif card.value == 'wild_draw4':
                    game.draw_cards(4)

                # Remove card from hand and return it
                return self.hand.pop(card_index)
        return None

    def choose_color(self):
        colors = ['red', 'yellow', 'green', 'blue']
        color_counts = {color: 0 for color in colors}
        for card in self.hand:
            if card.color in colors:
                color_counts[card.color] += 1

        return max(color_counts.items(), key=lambda x: x[1])[0]

    def has_playable_card(self, current_card):
        
        return any(card.matches(current_card) for card in self.hand)

class HumanPlayer(UnoPlayer):
    def play_turn(self, current_card, game):
        print("\nSpinner Rules: When you play a number card, the spinner activates with these possible outcomes:")
        print("1) Draw until red card drawn")
        print("2) Draw until blue card drawn")
        print("3) Remove all cards of same color")
        print("4) Cards War (compare highest numbers)")
        print("5) Swap cards with opponent")
        print("6) Show all your cards to opponent")

        print(f"\nCurrent card: {current_card}")
        print("Your hand:")
        for i, card in enumerate(self.hand):
            print(f"{i}: {card}")

        while True:
            try:
                choice = input("Enter card number to play, or 'd' to draw: ").strip().lower()
                if choice == 'd':
                    # Player chooses to draw
                    return None

                card_index = int(choice)
                if 0 <= card_index < len(self.hand):
                    card = self.hand[card_index]
                    if card.matches(current_card):
                        played_card = self.play_card(card_index, current_card, game)
                        if played_card is not None:
                            # Handle wild card color choice
                            if played_card.color == 'wild':
                                print("Choose a color (red, yellow, green, blue):")
                                color = input().strip().lower()
                                while color not in ['red', 'yellow', 'green', 'blue']:
                                    print("Invalid color. Choose from red, yellow, green, blue:")
                                    color = input().strip().lower()
                                played_card.chosen_color = color
                            return played_card
                        else:
                            print("Can't play that card. Try again.")
                    else:
                        print("That card doesn't match the current card. Try again.")
                else:
                    print("Invalid card number. Try again.")
            except ValueError:
                print("Invalid input. Enter a number or 'd' to draw.")

class AIPlayer(UnoPlayer):
    def play_turn(self, current_card, game):
        print(f"\n{self.name}'s turn...")
        time.sleep(1)  # Pause for realism

        # Find all playable cards
        playable_cards = [card for card in self.hand if card.matches(current_card)]

        if not playable_cards:
            # No playable cards, must draw
            print(f"{self.name} draws a card.")
            return None

        def card_score(card):
            if card.color == 'wild':
                if card.value == 'wild_draw4':
                    return 100  # Highest priority
                return 90
            elif card.value == 'draw2':
                return 80
            elif card.value == 'skip':
                return 70
            elif card.value == 'reverse':
                return 60
            elif isinstance(card.value, int):
                return card.value  # Higher numbers better
            return 0

        # Choose the best card to play
        best_card = max(playable_cards, key=card_score)
        card_index = self.hand.index(best_card)
        played_card = self.play_card(card_index, current_card, game)

        if played_card.color == 'wild':
            # AI chooses color strategically (most common color in hand)
            played_card.chosen_color = self.choose_color()
            print(f"{self.name} plays {played_card} and chooses {played_card.chosen_color}")
        else:
            print(f"{self.name} plays {played_card}")

        return played_card

class UnoGame:
    def __init__(self):
        self.players = [HumanPlayer("Player"), AIPlayer("AI")]
        self.current_player_index = 0
        self.direction = 1  # 1 for normal, -1 for reversed
        self.deck = []
        self.discard_pile = []
        self.spinner_outcomes = [
            "draw_until_red",
            "draw_until_blue",
            "remove_all_same_color",
            "cards_war",
            "swap_cards",
            "show_all_cards"
        ]
        self.initialize_deck()
        self.shuffle_deck()
        self.deal_cards()
        self.current_card = None
        self.start_game()

    def initialize_deck(self):
        colors = ['red', 'yellow', 'green', 'blue']
        numbers = list(range(10)) + list(range(1, 10))  # One 0, two 1-9 per color
        specials = ['skip', 'reverse', 'draw2'] * 2  # Two of each per color

        for color in colors:
            for num in numbers:
                self.deck.append(UnoCard(color, num))
            for spec in specials:
                self.deck.append(UnoCard(color, spec))

        # Add wild cards
        self.deck.extend([UnoCard('wild', 'wild')] * 4)
        self.deck.extend([UnoCard('wild', 'wild_draw4')] * 4)

    def shuffle_deck(self):
        random.shuffle(self.deck)

    def deal_cards(self):
        for _ in range(7):  # Deal 7 cards to each player
            for player in self.players:
                player.draw(self.deck.pop())

    def draw_card(self):
        if not self.deck:
            # Reshuffle discard pile (except top card) if deck is empty
            self.reshuffle_discard_pile()
        return self.deck.pop()

    def reshuffle_discard_pile(self):
        """Reshuffle discard pile into deck (keeping top card)"""
        print("Reshuffling discard pile into deck...")
        if len(self.discard_pile) <= 1:
            raise Exception("Not enough cards to reshuffle")

        top_card = self.discard_pile.pop()
        self.deck = self.discard_pile
        self.discard_pile = [top_card]
        self.shuffle_deck()
        print(f"New deck size: {len(self.deck)} cards")

    def start_game(self):
        # Flip the first card
        while True:
            card = self.draw_card()
            if card.color != 'wild':  # Can't start with wild
                self.discard_pile.append(card)
                self.current_card = card
                print(f"First card: {card}")
                break

    def next_player(self):
        self.current_player_index = (self.current_player_index + self.direction) % len(self.players)

    def skip_turn(self):
        print("Next player's turn is skipped!")
        self.next_player()

    def reverse_direction(self):
        print("Direction reversed!")
        self.direction *= -1

    def draw_cards(self, num):
        next_player_index = (self.current_player_index + self.direction) % len(self.players)
        print(f"{self.players[next_player_index].name} draws {num} cards!")
        for _ in range(num):
            self.players[next_player_index].draw(self.draw_card())

    def activate_spinner(self, played_card):
        """Handle spinner outcomes when a number card is played"""
        outcome = random.choice(self.spinner_outcomes)
        print(f"\n🎰 Spinner activated! Outcome: {outcome.replace('_', ' ').title()} 🎰")

        current_player = self.players[self.current_player_index]
        opponent_index = (self.current_player_index + self.direction) % len(self.players)
        opponent = self.players[opponent_index]

        if outcome == "draw_until_red":
            self.handle_draw_until_color(current_player, 'red')
        elif outcome == "draw_until_blue":
            self.handle_draw_until_color(current_player, 'blue')
        elif outcome == "remove_all_same_color":
            self.handle_remove_all_same_color(current_player, played_card.color)
        elif outcome == "cards_war":
            self.handle_cards_war(current_player, opponent)
        elif outcome == "swap_cards":
            self.handle_swap_cards(current_player, opponent)
        elif outcome == "show_all_cards":
            self.handle_show_all_cards(current_player, opponent)

    def handle_draw_until_color(self, player, target_color):
        """Player draws until they get a card of the specified color"""
        print(f"{player.name} draws until getting a {target_color} card...")
        drawn_cards = []
        while True:
            if not self.deck:
                self.reshuffle_discard_pile()

            card = self.draw_card()
            player.draw(card)
            drawn_cards.append(card)
            print(f"Drew: {card}")

            if card.color == target_color or (hasattr(card, 'chosen_color') and card.chosen_color == target_color):
                print(f"Found {target_color} card! Stopping.")
                break
            elif card.color == 'wild':
                # For wild cards, ask player to choose color (if human)
                if isinstance(player, HumanPlayer):
                    print("Choose a color for the wild card (red, yellow, green, blue):")
                    color = input().strip().lower()
                    while color not in ['red', 'yellow', 'green', 'blue']:
                        print("Invalid color. Choose from red, yellow, green, blue:")
                        color = input().strip().lower()
                    card.chosen_color = color
                else:
                    card.chosen_color = player.choose_color()

                if card.chosen_color == target_color:
                    print(f"Wild card set to {target_color}! Stopping.")
                    break

    def handle_remove_all_same_color(self, player, color):
        """Remove all cards of the specified color from player's hand"""
        print(f"Removing all {color} cards from {player.name}'s hand...")
        removed_cards = [card for card in player.hand if card.color == color]
        player.hand = [card for card in player.hand if card.color != color]
        self.discard_pile.extend(removed_cards)
        print(f"Removed {len(removed_cards)} {color} cards")

    def handle_cards_war(self, player1, player2):
        """Compare highest number cards - loser keeps both"""
        print("\n⚔️ Cards War! ⚔️")

        def get_highest_number_card(hand):
            number_cards = [card for card in hand if isinstance(card.value, int)]
            if not number_cards:
                return None
            return max(number_cards, key=lambda x: x.value)

        p1_card = get_highest_number_card(player1.hand)
        p2_card = get_highest_number_card(player2.hand)

        if not p1_card and not p2_card:
            print("No number cards found - war cancelled!")
            return

        print(f"{player1.name}'s highest: {p1_card}" if p1_card else f"{player1.name} has no number cards")
        print(f"{player2.name}'s highest: {p2_card}" if p2_card else f"{player2.name} has no number cards")

        if not p1_card or (p2_card and p1_card.value < p2_card.value):
            loser = player1
            winner = player2
        elif not p2_card or p1_card.value > p2_card.value:
            loser = player2
            winner = player1
        else:  # equal values
            print("Equal values - both players keep their cards")
            return

        print(f"{loser.name} loses the war and keeps both cards!")
        if p1_card:
            loser.hand.append(p1_card)
            player1.hand.remove(p1_card)
        if p2_card:
            loser.hand.append(p2_card)
            player2.hand.remove(p2_card)

    def handle_swap_cards(self, player1, player2):
        """Swap hands between two players"""
        print(f"\n🔄 {player1.name} and {player2.name} swap hands! 🔄")
        player1.hand, player2.hand = player2.hand, player1.hand

    def handle_show_all_cards(self, player, opponent):
        """Reveal player's hand to opponent"""
        print(f"\n👀 {player.name}'s hand is revealed to {opponent.name}! 👀")
        if isinstance(opponent, HumanPlayer):
            print(f"{player.name}'s cards:")
            for card in player.hand:
                print(f"  {card}")
        # For AI opponent, we would store this information in the AI's memory
        # but in this simple implementation, we'll just print it

    def play_turn(self):
        current_player = self.players[self.current_player_index]
        print(f"\n{'-'*30}\nIt's {current_player.name}'s turn")

        played_card = current_player.play_turn(self.current_card, self)

        if played_card is None:
            # Player chose to draw
            drawn_card = self.draw_card()
            current_player.draw(drawn_card)
            print(f"{current_player.name} drew a card")

            # Check if the drawn card can be played
            if drawn_card.matches(self.current_card):
                if isinstance(current_player, HumanPlayer):
                    print(f"You drew {drawn_card} which can be played!")
                    play_it = input("Do you want to play it? (y/n): ").strip().lower()
                    if play_it == 'y':
                        played_card = current_player.play_card(len(current_player.hand)-1, self.current_card, self)
                        if played_card.color == 'wild':
                            print("Choose a color (red, yellow, green, blue):")
                            color = input().strip().lower()
                            while color not in ['red', 'yellow', 'green', 'blue']:
                                print("Invalid color. Choose from red, yellow, green, blue:")
                                color = input().strip().lower()
                            played_card.chosen_color = color
                else:
                    # AI will always play a playable drawn card
                    played_card = current_player.play_card(len(current_player.hand)-1, self.current_card, self)
                    if played_card.color == 'wild':
                        played_card.chosen_color = current_player.choose_color()
                        print(f"{current_player.name} plays {played_card} and chooses {played_card.chosen_color}")
                    else:
                        print(f"{current_player.name} plays the drawn card: {played_card}")

        if played_card is not None:
            self.discard_pile.append(played_card)
            self.current_card = played_card

            # Activate spinner if a number card was played
            if isinstance(played_card.value, int):
                self.activate_spinner(played_card)

            # Check for win condition
            if len(current_player.hand) == 0:
                print(f"\n{'*'*40}")
                print(f"* {current_player.name} has won the game! *")
                print(f"{'*'*40}")
                return True

        self.next_player()
        return False

    def play_game(self):
        while True:
            game_over = self.play_turn()
            if game_over:
                break

if __name__ == "__main__":
    print("Welcome to UNO with Spinner!")
    print("Rules:")
    print("- Match the current card by color or number/symbol")
    print("- Wild cards can be played on anything (you choose the color)")
    print("- Action cards have special effects (skip, reverse, draw 2/4)")
    print("- When you play a number card, the spinner activates with random effects!")
    print("- First to empty their hand wins!")
    print("- Enter 'd' to draw a card when it's your turn")
    print("\nStarting game...")

    game = UnoGame()
    game.play_game()
