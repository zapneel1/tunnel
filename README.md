import json
import random
import time
from pathlib import Path


class Entity:
    """Base class for characters and enemies in the game."""

    def __init__(self, name: str, health: int, max_health: int, attack_power: int):
        self.name = name
        self.health = health
        self.max_health = max_health
        self.attack_power = attack_power

    def is_alive(self) -> bool:
        return self.health > 0

    def take_damage(self, damage: int) -> int:
        actual_damage = max(0, damage)
        self.health = max(0, self.health - actual_damage)
        return actual_damage


class Player(Entity):
    """The user-controlled hero character."""

    def __init__(self, name: str):
        super().__init__(name=name, health=100, max_health=100, attack_power=15)
        self.gold = 50
        self.inventory = ["Health Potion"]
        self.experience = 0
        self.level = 1

    def use_potion(self) -> bool:
        if "Health Potion" in self.inventory:
            self.inventory.remove("Health Potion")
            heal_amount = 40
            self.health = min(self.max_health, self.health + heal_amount)
            print(f"\n✨ You drank a Health Potion and recovered {heal_amount} HP!")
            return True
        print("\n❌ You don't have any Health Potions left!")
        return False

    def gain_xp(self, amount: int):
        self.experience += amount
        print(f"🔸 Gained {amount} experience points.")
        xp_needed = self.level * 50
        if self.experience >= xp_needed:
            self.level_up()

    def level_up(self):
        self.level += 1
        self.experience = 0
        self.max_health += 20
        self.health = self.max_health
        self.attack_power += 5
        print(f"\n🎉 LEVEL UP! You reached Level {self.level}!")
        print(f"❤️ Max HP increased to {self.max_health} | ⚔️ Attack rose to {self.attack_power}!")

    def to_dict(self) -> dict:
        return {
            "name": self.name,
            "health": self.health,
            "max_health": self.max_health,
            "attack_power": self.attack_power,
            "gold": self.gold,
            "inventory": self.inventory,
            "experience": self.experience,
            "level": self.level,
        }

    def load_dict(self, data: dict):
        self.health = data["health"]
        self.max_health = data["max_health"]
        self.attack_power = data["attack_power"]
        self.gold = data["gold"]
        self.inventory = data["inventory"]
        self.experience = data["experience"]
        self.level = data["level"]


class GameEngine:
    """Manages game state, loops, combat, and saving/loading mechanics."""

    SAVE_FILE = Path("rpg_save.json")

    def __init__(self):
        self.player = None
        self.monsters = [
            {"name": "Goblin Thief", "health": 40, "attack": 8, "xp": 20, "gold": 15},
            {"name": "Feral Wolf", "health": 55, "attack": 12, "xp": 35, "gold": 25},
            {"name": "Shadow Knight", "health": 90, "attack": 18, "xp": 70, "gold": 60},
        ]

    def slow_print(self, text: str, delay: float = 0.02):
        for char in text:
            print(char, end="", flush=True)
            time.sleep(delay)
        print()

    def main_menu(self):
        print("=" * 40)
        print("         LEGEND OF PYTHONIA RPG        ")
        print("=" * 40)
        print("1. Start New Adventure")
        print("2. Load Saved Game")
        print("3. Exit Game")
        print("=" * 40)

        choice = input("Select an option (1-3): ").strip()
        if choice == "1":
            name = input("Enter your hero's name: ").strip() or "Unknown Hero"
            self.player = Player(name)
            self.town_loop()
        elif choice == "2":
            if self.load_game():
                self.town_loop()
            else:
                self.main_menu()
        else:
            print("\nGoodbye, traveler!")

    def save_game(self):
        try:
            with open(self.SAVE_FILE, "w") as f:
                json.dump(self.player.to_dict(), f, indent=4)
            print("\n💾 Progress saved successfully!")
        except Exception as e:
            print(f"\n❌ Error saving file: {e}")

    def load_game(self) -> bool:
        if not self.SAVE_FILE.exists():
            print("\n❌ No saved game data found!")
            return False
        try:
            with open(self.SAVE_FILE, "r") as f:
                data = json.load(f)
            self.player = Player(data["name"])
            self.player.load_dict(data)
            print(f"\n✨ Welcome back, {self.player.name}!")
            return True
        except Exception as e:
            print(f"\n❌ Error loading file: {data}")
            return False

    def town_loop(self):
        while self.player.is_alive():
            print(f"\n--- TOWN SQUARE [Level {self.player.level}] ---")
            print(f"Hero: {self.player.name} | HP: {self.player.health}/{self.player.max_health}")
            print(f"Wallet: {self.player.gold} Gold | Items: {self.player.inventory}")
            print("-" * 30)
            print("1. Venture into the Dungeon (Battle)")
            print("2. Visit the Alchemist Shop")
            print("3. Save Game")
            print("4. Rest at the Inn (Heal for 15 Gold)")
            print("5. Quit to Main Menu")

            action = input("What will you do? (1-5): ").strip()

            if action == "1":
                self.combat_loop()
            elif action == "2":
                self.shop_loop()
            elif action == "3":
                self.save_game()
            elif action == "4":
                if self.player.gold >= 15:
                    self.player.gold -= 15
                    self.player.health = self.player.max_health
                    print("\n💤 You slept like a log. Health fully restored!")
                else:
                    print("\n❌ You don't have enough gold to stay at the Inn!")
            elif action == "5":
                break
            else:
                print("\n❌ Invalid choice! Look around carefully.")

    def shop_loop(self):
        while True:
            print(f"\n--- ALCHEMIST SHOP (Your Gold: {self.player.gold}) ---")
            print("1. Buy Health Potion (20 Gold)")
            print("2. Leave Shop")

            choice = input("Select an option (1-2): ").strip()
            if choice == "1":
                if self.player.gold >= 20:
                    self.player.gold -= 20
                    self.player.inventory.append("Health Potion")
                    print("\n🧪 Purchased 1 Health Potion.")
                else:
                    print("\n❌ Not enough gold!")
            elif choice == "2":
                break

    def combat_loop(self):
        monster_data = random.choice(self.monsters)
        monster = Entity(
            name=monster_data["name"],
            health=monster_data["health"],
            max_health=monster_data["health"],
            attack_power=monster_data["attack"],
        )

        self.slow_print(f"\n💥 A wild {monster.name} emerges from the dark!")

        while monster.is_alive() and self.player.is_alive():
            print(f"\n⚔️ {self.player.name}: {self.player.health}/{self.player.max_health} HP")
            print(f"👹 {monster.name}: {monster.health}/{monster.max_health} HP")
            print("1. Attack")
            print("2. Use Potion")
            print("3. Run Away")

            choice = input("Your move: ").strip()

            if choice == "1":
                p_dmg = random.randint(self.player.attack_power - 3, self.player.attack_power + 3)
                inflicted = monster.take_damage(p_dmg)
                print(f"\n⚔️ You strike the {monster.name} for {inflicted} damage!")

                if monster.is_alive():
                    m_dmg = random.randint(monster.attack_power - 2, monster.attack_power + 2)
                    taken = self.player.take_damage(m_dmg)
                    print(f"💥 {monster.name} claws you for {taken} damage!")

            elif choice == "2":
                if self.player.use_potion() and monster.is_alive():
                    m_dmg = random.randint(monster.attack_power - 2, monster.attack_power + 2)
                    taken = self.player.take_damage(m_dmg)
                    print(f"💥 {monster.name} took advantage of your distraction and hit you for {taken} damage!")

            elif choice == "3":
                if random.random() < 0.5:
                    print("\n💨 You successfully escaped back to town!")
                    return
                print("\n❌ You tripped while running away! The battle continues!")
                m_dmg = random.randint(monster.attack_power - 2, monster.attack_power + 2)
                taken = self.player.take_damage(m_dmg)
                print(f"💥 {monster.name} strikes you from behind for {taken} damage!")

        if self.player.is_alive():
            print(f"\n🏆 You defeated the {monster.name}!")
            self.player.gold += monster_data["gold"]
            print(f"💰 Looted {monster_data['gold']} gold pieces.")
            self.player.gain_xp(monster_data["xp"])
        else:
            print("\n💀 You fell in battle... GAME OVER.")


if __name__ == "__main__":
    game = GameEngine()
    game.main_menu()
