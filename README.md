from kivy.app import App
from kivy.uix.widget import Widget
from kivy.properties import NumericProperty, ListProperty
from kivy.clock import Clock
from kivy.core.window import Window
from kivy.uix.label import Label
from kivy.uix.boxlayout import BoxLayout
import random

Window.size = (400, 600)

class Bird(Widget):
    """Bird object for the game"""
    velocity = NumericProperty(0)
    
    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        self.size = (40, 40)
        self.pos = (50, 300)
        self.gravity = 0.6
        self.jump_velocity = 15
    
    def jump(self):
        """Make the bird jump"""
        self.velocity = self.jump_velocity
    
    def update(self, dt):
        """Update bird position and velocity"""
        # Apply gravity
        self.velocity -= self.gravity
        
        # Update position
        self.y += self.velocity
        
        # Keep bird within screen bounds (top and bottom)
        if self.y < 0:
            self.y = 0
            self.velocity = 0
        if self.y + self.height > self.parent.height:
            self.y = self.parent.height - self.height


class Pipe(Widget):
    """Pipe obstacle"""
    def __init__(self, x, gap_y, **kwargs):
        super().__init__(**kwargs)
        self.x = x
        self.gap_y = gap_y
        self.width = 50
        self.gap_size = 150
        self.speed = 4
        self.scored = False
    
    def update(self, dt):
        """Move pipe to the left"""
        self.x -= self.speed


class FlappyBirdGame(Widget):
    """Main game widget"""
    score = NumericProperty(0)
    
    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        self.bird = Bird()
        self.add_widget(self.bird)
        
        self.pipes = []
        self.game_over = False
        self.pipe_spawn_counter = 0
        self.pipe_spawn_interval = 100
        
        # Bind touch events for jumping
        self.bind(on_touch_down=self.on_touch_down)
        
        # Start game loop
        Clock.schedule_interval(self.update, 1.0 / 60.0)  # 60 FPS
    
    def on_touch_down(self, touch):
        """Handle tap/click to make bird jump"""
        if not self.game_over:
            self.bird.jump()
        else:
            self.reset_game()
        return True
    
    def spawn_pipe(self):
        """Create a new pipe obstacle"""
        gap_y = random.randint(100, self.height - 250)
        pipe = Pipe(self.width, gap_y)
        self.pipes.append(pipe)
        self.add_widget(pipe)
    
    def reset_game(self):
        """Reset the game to initial state"""
        self.score = 0
        self.game_over = False
        self.pipes = []
        self.pipe_spawn_counter = 0
        
        # Clear old pipes from widget tree
        for child in list(self.children):
            if isinstance(child, Pipe):
                self.remove_widget(child)
        
        # Reset bird position
        self.bird.pos = (50, 300)
        self.bird.velocity = 0
    
    def check_collision(self):
        """Check if bird collides with pipes or goes out of bounds"""
        bird_x = self.bird.x
        bird_y = self.bird.y
        bird_w = self.bird.width
        bird_h = self.bird.height
        
        for pipe in self.pipes:
            pipe_x = pipe.x
            pipe_w = pipe.width
            gap_y = pipe.gap_y
            gap_size = pipe.gap_size
            
            # Check if bird is at pipe's x position
            if bird_x + bird_w > pipe_x and bird_x < pipe_x + pipe_w:
                # Check if bird hit top or bottom pipe
                if bird_y < gap_y or bird_y + bird_h > gap_y + gap_size:
                    return True
                
                # Award score when passing pipe
                if not pipe.scored and bird_x > pipe_x + pipe_w:
                    pipe.scored = True
                    self.score += 1
        
        return False
    
    def update(self, dt):
        """Update game state"""
        if self.game_over:
            return
        
        # Update bird physics
        self.bird.update(dt)
        
        # Spawn pipes
        self.pipe_spawn_counter += 1
        if self.pipe_spawn_counter >= self.pipe_spawn_interval:
            self.spawn_pipe()
            self.pipe_spawn_counter = 0
        
        # Update pipes
        for pipe in self.pipes[:]:
            pipe.update(dt)
            
            # Remove pipes that are off screen
            if pipe.x + pipe.width < 0:
                self.remove_widget(pipe)
                self.pipes.remove(pipe)
        
        # Check collisions
        if self.check_collision():
            self.game_over = True
    
    def on_size(self, *args):
        """Handle window resize"""
        pass


class FlappyBirdApp(App):
    def build(self):
        """Build the app"""
        layout = BoxLayout(orientation='vertical')
        
        # Score label
        self.score_label = Label(size_hint_y=0.1, font_size='30sp', text='Score: 0')
        layout.add_widget(self.score_label)
        
        # Game area
        self.game = FlappyBirdGame(size_hint_y=0.9)
        self.game.bind(score=self.update_score_label)
        layout.add_widget(self.game)
        
        return layout
    
    def update_score_label(self, instance, value):
        """Update the score label"""
        self.score_label.text = f'Score: {int(value)}'


if __name__ == '__main__':
    FlappyBirdApp().run()
