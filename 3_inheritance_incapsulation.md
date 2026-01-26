```python
from __future__ import annotations

#батарейка - базовый класс
class Battery:
    def __init__(self, capacity_mah: int, nominal_voltage_v: float, mass_kg: float):
        self.__capacity_mah = capacity_mah
        self.__nominal_voltage_v = nominal_voltage_v
        self.__mass_kg = mass_kg
        self.__current_voltage_v = nominal_voltage_v
        self.__is_low = False
        self.low_threshold_v = 700

    @property
    def capacity_mah(self) -> int:
        return self.__capacity_mah

    @property
    def nominal_voltage_v(self) -> float:
        return self.__nominal_voltage_v

    @property
    def mass_kg(self) -> float:
        return self.__mass_kg

    @property
    def current_voltage_v(self) -> float:
        return self.__current_voltage_v

    @property
    def is_low(self) -> bool:
        return self.__is_low

    def update_voltage(self, voltage_v: float) -> None:
        self.__current_voltage_v = float(voltage_v)

    def update_low_flag(self, low_threshold_v: float) -> None:
        self.__is_low = self.__current_voltage_v < float(low_threshold_v)

    def estimate_energy_wh(self) -> float:
        return (self.__capacity_mah / 1000.0) * self.__nominal_voltage_v

    def __str__(self) -> str:
        return (f"{self.__class__.__name__}(capacity={self.__capacity_mah}mAh, "
                f"V={self.__current_voltage_v:.2f}V, low={self.__is_low})")


class LiIon18650(Battery):
    def __init__(self, capacity_mah: int = 3500, current_voltage_v: float = 4.10):
        super().__init__(capacity_mah=capacity_mah, nominal_voltage_v=3.60, mass_kg=0.045)
        self.__max_discharge_a = 10.0
        self.update_voltage(current_voltage_v)

    def recommended_charge_current_a(self) -> float:
        return (self.capacity_mah / 1000.0) * 0.5


    def max_continuous_power_w(self) -> float:
        return self.current_voltage_v * self.__max_discharge_a

    def __str__(self) -> str:
        return (super().__str__()[:-1] +
                f", max_discharge={self.__max_discharge_a}A)")


class LiPo4S(Battery):
    def __init__(self, capacity_mah: int = 5200):
        # 4S: nominal ~ 14.8V
        super().__init__(capacity_mah=capacity_mah, nominal_voltage_v=14.80, mass_kg=0.500)
        self.__cells = 4
        self.__cell_voltages_v = [3.70, 3.70, 3.70, 3.70]
        self.update_voltage(sum(self.__cell_voltages_v))


    def balance_cells(self) -> None:
        avg = sum(self.__cell_voltages_v) / self.__cells
        self.__cell_voltages_v = [avg] * self.__cells
        self.update_voltage(sum(self.__cell_voltages_v))


    def set_storage_mode(self) -> None:
        self.__cell_voltages_v = [3.85] * self.__cells
        self.update_voltage(sum(self.__cell_voltages_v))

    def __str__(self) -> str:
        cells = ",".join(f"{v:.2f}" for v in self.__cell_voltages_v)
        return super().__str__()[:-1] + f", cells=[{cells}])"

class Motor:
    def __init__(self, mass_kg: float, engine_power_w: float):
        self.__mass_kg = float(mass_kg)
        self.__engine_power_w = float(engine_power_w)
        self.__is_running = False

    @property
    def is_running(self) -> bool:
        return self.__is_running

    def start(self) -> None:
        self.__is_running = True

    def stop(self) -> None:
        self.__is_running = False

    def __str__(self) -> str:
        return (f"Motor(power={self.__engine_power_w}W, mass={self.__mass_kg}kg, "
                f"running={self.__is_running})")


class Machine:
    def __init__(self, vehicle_type: str, mass_kg: float, battery: Battery, motor: Motor):
        self.__vehicle_type = str(vehicle_type)
        self.__mass_kg = float(mass_kg)
        self.__battery = battery
        self.__motor = motor
        self.__mode = "MANUAL"
        self.__armed = False

    @property
    def vehicle_type(self) -> str:
        return self.__vehicle_type

    @property
    def mass_kg(self) -> float:
        return self.__mass_kg

    @property
    def mode(self) -> str:
        return self.__mode

    @property
    def armed(self) -> bool:
        return self.__armed

    @property
    def battery(self) -> Battery:
        return self.__battery

    @property
    def motor(self) -> Motor:
        return self.__motor

    def set_mode(self, mode: str) -> None:
        self.__mode = str(mode)

    def arm(self) -> None:

        self.__battery.update_low_flag(low_threshold_v=self.__low_voltage_threshold())
        if not self.__battery.is_low:
            self.__armed = True

    def disarm(self) -> None:
        self.__armed = False
        self.__motor.stop()

    def start(self) -> None:

        if not self.__armed:
            return
        self.__battery.update_low_flag(low_threshold_v=self.__low_voltage_threshold())
        if self.__battery.is_low:
            self.disarm()
            return
        self.__motor.start()

    def __low_voltage_threshold(self) -> float:

        if isinstance(self.__battery, LiPo4S):
            return 14.0
        return 3.5

    def __str__(self) -> str:
        return (f"{self.__class__.__name__}(type={self.__vehicle_type}, mass={self.__mass_kg}kg, "
                f"mode={self.__mode}, armed={self.__armed}, battery={self.__battery}, motor={self.__motor})")


class Rover(Machine):
    def __init__(self, mass_kg: float, battery: Battery, motor: Motor, num_wheels: int):
        super().__init__(vehicle_type="rover", mass_kg=mass_kg, battery=battery, motor=motor)
        self.__num_wheels = int(num_wheels)
        self.__diff_lock_enabled = False

    def enable_diff_lock(self) -> None:
        self.__diff_lock_enabled = True


    def set_wheel_count(self, num_wheels: int) -> None:
        if num_wheels < 2:
            return
        self.__num_wheels = int(num_wheels)

    def __str__(self) -> str:
        base = super().__str__()[:-1]
        return base + f", wheels={self.__num_wheels}, diff_lock={self.__diff_lock_enabled})"


class Copter(Machine):
    def __init__(self, mass_kg: float, battery: Battery, motor: Motor, num_rotors: int):
        super().__init__(vehicle_type="copter", mass_kg=mass_kg, battery=battery, motor=motor)
        self.__num_rotors = int(num_rotors)
        self.__altitude_hold_enabled = False


    def enable_altitude_hold(self) -> None:
        self.__altitude_hold_enabled = True


    def emergency_land(self) -> None:

        self.disarm()

    def __str__(self) -> str:
        base = super().__str__()[:-1]
        return base + f", rotors={self.__num_rotors}, alt_hold={self.__altitude_hold_enabled})"


def main() -> None:
    cell = LiIon18650(capacity_mah=3500, current_voltage_v=3.95)
    pack = LiPo4S(capacity_mah=5200)

    rover_motor = Motor(mass_kg=1.2, engine_power_w=15.0)
    copter_motor = Motor(mass_kg=0.8, engine_power_w=250.0)

    rover = Rover(mass_kg=8.5, battery=cell, motor=rover_motor, num_wheels=6)
    copter = Copter(mass_kg=1.6, battery=pack, motor=copter_motor, num_rotors=4)
    rover.set_mode("AUTO")
    rover.enable_diff_lock()
    rover.set_wheel_count(4)
    copter.set_mode("LOITER")
    copter.enable_altitude_hold()

    cell.update_voltage(3.40)   
    pack.set_storage_mode()    
    pack.balance_cells()       

    rover.arm()
    rover.start()             
    copter.arm()
    copter.start()


    print(rover)
    print(copter)
    print("LiIon18650 recommended_charge_current_a:", f"{cell.recommended_charge_current_a():.2f}A")
    print("LiIon18650 max_continuous_power_w:", f"{cell.max_continuous_power_w():.1f}W")
    copter.emergency_land()
    print("После emergency_land():", copter)


if __name__ == "__main__":
    main()

```
