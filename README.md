# ESP32 DevKit V1 - Vehicle Horn Detection & Obstacle Avoidance
# MicroPython Code - Flash this to your ESP32
# Hardware: HC-SR04, Analog Microphone, 2x Coin Vibrators

import machine
import time
import array
import utime
from machine import Pin, PWM, ADC
import math

# ==================== CONFIGURATION ====================

# Pin Definitions
TRIG_PIN = 5          # HC-SR04 Trigger
ECHO_PIN = 18         # HC-SR04 Echo
MIC_PIN = 34          # Analog Microphone (ADC1_CH6)
VIBRATOR_1_PIN = 25   # Left/First Vibrator
VIBRATOR_2_PIN = 26   # Right/Second Vibrator

# Detection Thresholds
OBSTACLE_WARNING_DISTANCE = 100  # cm - warning zone
OBSTACLE_CRITICAL_DISTANCE = 30  # cm - critical zone
AUDIO_CHECK_INTERVAL = 200       # ms - how often to check audio
DETECTION_COOLDOWN = 500         # ms - minimum time between feedbacks

# Audio Sampling
SAMPLE_RATE = 8000               # Hz - audio sample rate
SAMPLE_DURATION_MS = 150         # ms - audio capture duration
ADC_CENTER = 2048                # 12-bit ADC center value

# Haptic Feedback Intensity (0-100)
HAPTIC_INTENSITY_LOW = 30        # Subtle feedback
HAPTIC_INTENSITY_MEDIUM = 40     # Medium feedback
HAPTIC_INTENSITY_HIGH = 50       # Strong feedback

# ==================== HARDWARE CONTROLLER ====================

class HardwareController:
    """Controls all hardware components"""
    
    def __init__(self):
        print("Initializing hardware...")
        
        # Ultrasonic sensor setup
        self.trig = Pin(TRIG_PIN, Pin.OUT)
        self.echo = Pin(ECHO_PIN, Pin.IN)
        self.trig.value(0)
        
        # Microphone ADC setup
        self.mic = ADC(Pin(MIC_PIN))
        self.mic.atten(ADC.ATTN_11DB)    # 0-3.6V range
        self.mic.width(ADC.WIDTH_12BIT)  # 12-bit resolution (0-4095)
        
        # Vibrator PWM setup (200Hz for smooth haptic feel)
        self.vibrator1 = PWM(Pin(VIBRATOR_1_PIN), freq=200)
        self.vibrator2 = PWM(Pin(VIBRATOR_2_PIN), freq=200)
        self.vibrator1.duty(0)
        self.vibrator2.duty(0)
        
        print("✓ Hardware initialized")
    
    def measure_distance(self):
        """
        Measure distance using HC-SR04 ultrasonic sensor
        Returns: distance in cm, or None if timeout
        """
        # Send 10us pulse
        self.trig.value(0)
        utime.sleep_us(2)
        self.trig.value(1)
        utime.sleep_us(10)
        self.trig.value(0)
        
        # Measure echo duration with timeout
        timeout_us = 30000  # 30ms timeout
        start_time = utime.ticks_us()
        
        # Wait for echo start
        while self.echo.value() == 0:
            if utime.ticks_diff(utime.ticks_us(), start_time) > timeout_us:
                return None
        pulse_start = utime.ticks_us()
        
        # Wait for echo end
        while self.echo.value() == 1:
            if utime.ticks_diff(utime.ticks_us(), start_time) > timeout_us:
                return None
        pulse_end = utime.ticks_us()
        
        # Calculate distance (speed of sound = 343 m/s)
        pulse_duration = utime.ticks_diff(pulse_end, pulse_start)
        distance = (pulse_duration * 0.0343) / 2
        
        return distance
    
    def read_audio_samples(self, duration_ms=SAMPLE_DURATION_MS, sample_rate=SAMPLE_RATE):
        """
        Capture audio samples from microphone
        Returns: list of ADC readings
        """
        samples = []
        sample_interval_us = 1000000 // sample_rate
        num_samples = (duration_ms * sample_rate) // 1000
        
        for _ in range(num_samples):
            start = utime.ticks_us()
            samples.append(self.mic.read())
            
            # Maintain sample rate
            elapsed = utime.ticks_diff(utime.ticks_us(), start)
            if elapsed < sample_interval_us:
                utime.sleep_us(sample_interval_us - elapsed)
        
        return samples
    
    def haptic_pulse(self, pattern='single', intensity=30):
        """
        Generate haptic feedback patterns
        
        Args:
            pattern: 'single', 'double', 'alternating', 'warning'
            intensity: 0-100 (percentage of max vibration)
        """
        duty = int((intensity / 100) * 1023)  # Convert to 10-bit PWM duty
        
        if pattern == 'single':
            # Single short pulse - for regular obstacles
            self._vibrate_both(duty, 50)
            
        elif pattern == 'double':
            # Double pulse - for bike horns (higher urgency)
            self._vibrate_both(duty, 50)
            utime.sleep_ms(100)
            self._vibrate_both(duty, 50)
            
        elif pattern == 'alternating':
            # Alternating vibrators - for car/truck horns
            self._vibrate_left(duty, 80)
            utime.sleep_ms(30)
            self._vibrate_right(duty, 80)
            
        elif pattern == 'warning':
            # Rapid pulses - critical obstacles
            for _ in range(3):
                self._vibrate_both(duty, 30)
                utime.sleep_ms(30)
        
        # Ensure vibrators are off
        self.stop_vibration()
    
    def _vibrate_both(self, duty, duration_ms):
        """Vibrate both motors"""
        self.vibrator1.duty(duty)
        self.vibrator2.duty(duty)
        utime.sleep_ms(duration_ms)
        self.stop_vibration()
    
    def _vibrate_left(self, duty, duration_ms):
        """Vibrate left motor only"""
        self.vibrator1.duty(duty)
        utime.sleep_ms(duration_ms)
        self.vibrator1.duty(0)
    
    def _vibrate_right(self, duty, duration_ms):
        """Vibrate right motor only"""
        self.vibrator2.duty(duty)
        utime.sleep_ms(duration_ms)
        self.vibrator2.duty(0)
    
    def stop_vibration(self):
        """Stop all vibrations"""
        self.vibrator1.duty(0)
        self.vibrator2.duty(0)
    
    def cleanup(self):
        """Clean up hardware on exit"""
        self.stop_vibration()
        self.vibrator1.deinit()
        self.vibrator2.deinit()

# ==================== FEATURE EXTRACTION ====================

class AudioFeatureExtractor:
    """Extract features from audio samples for classification"""
    
    @staticmethod
    def extract_features(samples):
        """
        Extract audio features from raw ADC samples
        Returns: list of features [rms, zcr, peak, spectral_flux, mean]
        """
        n = len(samples)
        if n == 0:
            return [0, 0, 0, 0, 0]
        
        # Convert to array for faster processing
        samples_array = array.array('i', samples)
        
        # Calculate mean (DC offset)
        mean_val = sum(samples_array) / n
        
        # Center samples around zero
        centered = [x - mean_val for x in samples_array]
        
        # RMS Energy (Root Mean Square)
        energy = sum(x*x for x in centered) / n
        rms = math.sqrt(energy) if energy > 0 else 0
        
        # Zero Crossing Rate
        zero_crossings = 0
        for i in range(n - 1):
            if centered[i] * centered[i+1] < 0:
                zero_crossings += 1
        zcr = zero_crossings / n
        
        # Peak amplitude
        peak = max(abs(x) for x in centered)
        
        # Spectral flux (measure of high-frequency content)
        # Count rapid changes (approximation of spectral content)
        spectral_flux = 0
        threshold = peak * 0.2  # 20% of peak
        for i in range(n - 1):
            diff = abs(samples_array[i+1] - samples_array[i])
            if diff > threshold:
                spectral_flux += 1
        
        return [rms, zcr, peak, spectral_flux, mean_val]

# ==================== HORN CLASSIFIER ====================

class HornClassifier:
    """
    Rule-based classifier for vehicle horn detection
    Optimized for ESP32 without TFLite
    """
    
    def __init__(self):
        # Classification thresholds (tune these based on your microphone)
        self.min_rms = 100          # Minimum energy for horn
        self.min_peak = 200         # Minimum peak amplitude
        self.zcr_range = (0.05, 0.6) # Zero crossing rate range
        
        # Horn type characteristics
        self.bike_high_freq_threshold = 25
        self.truck_peak_threshold = 1500
        
    def classify(self, features):
        """
        Classify audio based on extracted features
        
        Args:
            features: [rms, zcr, peak, spectral_flux, mean]
        
        Returns:
            (is_horn, horn_type, confidence)
            horn_type: 0=none, 1=car, 2=bike, 3=truck
            confidence: 0.0-1.0
        """
        rms, zcr, peak, spectral_flux, mean = features
        
        # Check if sound is loud enough to be a horn
        if rms < self.min_rms or peak < self.min_peak:
            return (False, 0, 0.0)
        
        # Check zero crossing rate (horns have characteristic frequency patterns)
        if not (self.zcr_range[0] <= zcr <= self.zcr_range[1]):
            return (False, 0, 0.0)
        
        # Calculate confidence based on signal strength
        confidence = min(rms / 1000, 1.0)
        
        # Classify horn type based on spectral and amplitude characteristics
        
        # Bike horns: High frequency content, moderate amplitude
        if spectral_flux > self.bike_high_freq_threshold and peak < 1200:
            return (True, 2, confidence)
        
        # Truck horns: Very loud, lower frequency
        elif peak > self.truck_peak_threshold:
            return (True, 3, confidence)
        
        # Car horns: Medium frequency and amplitude
        else:
            return (True, 1, confidence)

# ==================== MAIN APPLICATION ====================

class AssistiveDevice:
    """Main application controller"""
    
    def __init__(self):
        print("\n" + "="*50)
        print("ESP32 Assistive Device Starting...")
        print("="*50)
        
        self.hardware = HardwareController()
        self.feature_extractor = AudioFeatureExtractor()
        self.classifier = HornClassifier()
        
        # Timing control
        self.last_feedback_time = 0
        self.last_audio_check = 0
        
        # Detection state
        self.obstacle_detected = False
        self.horn_detected = False
        
        print("✓ System ready!")
        print("="*50 + "\n")
    
    def check_obstacles(self):
        """
        Check for obstacles using ultrasonic sensor
        Returns: (level, distance) where level is 'clear', 'warning', or 'critical'
        """
        distance = self.hardware.measure_distance()
        
        if distance is None:
            return ('clear', None)
        
        if distance < OBSTACLE_CRITICAL_DISTANCE:
            return ('critical', distance)
        elif distance < OBSTACLE_WARNING_DISTANCE:
            return ('warning', distance)
        else:
            return ('clear', distance)
    
    def check_audio(self):
        """
        Capture and analyze audio for horn detection
        Returns: (is_horn, horn_type, confidence)
        """
        # Capture audio
        samples = self.hardware.read_audio_samples()
        
        # Extract features
        features = self.feature_extractor.extract_features(samples)
        
        # Classify
        is_horn, horn_type, confidence = self.classifier.classify(features)
        
        return is_horn, horn_type, confidence
    
    def provide_feedback(self, obstacle_level, horn_detected, horn_type, confidence):
        """
        Provide appropriate haptic feedback based on detections
        Priority: Critical obstacle > Horn detection > Warning obstacle
        """
        current_time = utime.ticks_ms()
        
        # Cooldown check to avoid feedback spam
        if utime.ticks_diff(current_time, self.last_feedback_time) < DETECTION_COOLDOWN:
            return
        
        feedback_given = False
        
        # PRIORITY 1: Critical obstacle (most urgent)
        if obstacle_level == 'critical':
            self.hardware.haptic_pulse('warning', HAPTIC_INTENSITY_HIGH)
            print("⚠ CRITICAL OBSTACLE!")
            feedback_given = True
        
        # PRIORITY 2: Horn detection
        elif horn_detected and confidence > 0.3:
            horn_names = ['None', 'Car', 'Bike', 'Truck']
            
            if horn_type == 2:  # Bike horn (higher urgency)
                self.hardware.haptic_pulse('double', HAPTIC_INTENSITY_MEDIUM)
            else:  # Car or truck horn
                self.hardware.haptic_pulse('alternating', HAPTIC_INTENSITY_LOW)
            
            print(f"🔊 {horn_names[horn_type]} horn detected! (confidence: {confidence:.2f})")
            feedback_given = True
        
        # PRIORITY 3: Warning obstacle
        elif obstacle_level == 'warning':
            self.hardware.haptic_pulse('single', HAPTIC_INTENSITY_LOW)
            print("⚠ Obstacle detected")
            feedback_given = True
        
        if feedback_given:
            self.last_feedback_time = current_time
    
    def run(self):
        """Main detection loop"""
        print("Starting detection loop...")
        print("Monitoring for obstacles and vehicle horns\n")
        
        try:
            while True:
                current_time = utime.ticks_ms()
                
                # Check obstacles continuously (fast)
                obstacle_level, distance = self.check_obstacles()
                
                # Check audio periodically (slower, more resource intensive)
                horn_detected = False
                horn_type = 0
                confidence = 0.0
                
                if utime.ticks_diff(current_time, self.last_audio_check) >= AUDIO_CHECK_INTERVAL:
                    horn_detected, horn_type, confidence = self.check_audio()
                    self.last_audio_check = current_time
                
                # Provide appropriate feedback
                self.provide_feedback(obstacle_level, horn_detected, horn_type, confidence)
                
                # Small delay to prevent CPU overload
                utime.sleep_ms(50)
                
        except KeyboardInterrupt:
            print("\n\nStopping device...")
            self.hardware.cleanup()
            print("Device stopped safely")
            
        except Exception as e:
            print(f"\n❌ Error: {e}")
            self.hardware.cleanup()
            print("Device stopped due to error")

# ==================== ENTRY POINT ====================

def main():
    """Main entry point"""
    device = AssistiveDevice()
    device.run()

# Start the application
if __name__ == '__main__':
    main()
