1. Структура проекта:
```txt
OOAP_modules/
  main.py
  power/
    __init__.py
    batteries.py
  autopilot/
    __init__.py
    machines.py
    diagnostics.py
```
2. Файл с батарейками:
```python
class Battery:
    def __init__(self, capacity_mah: int, nominal_v: float):
        self.__capacity_mah = int(capacity_mah)
        self.__nominal_v = float(nominal_v)
        self.__current_v = float(nominal_v)

    @property
    def capacity_mah(self) -> int:
        return self.__capacity_mah
    
    @property
    def nominal_v(self) -> float:
        return self.__nominal_v
    
    @property
    def current_v(self) -> float:
        return self.__current_v
    
    def set_voltage(self, v: float) -> None:
        self.__current_v = float(v)

    def is_low(self) -> bool:
        return self.__current_v < (0.9 * self.__nominal_v)
    
    def __str__(self) -> str:
        return f"{self.__class__.__name__} (V={self.__current_v:.2f}, cap={self.__capacity_mah}mAh)"
    
class LiIon18659(Battery):
    def __init__(self, capacity_mah: int = 3500):
        super().__init__(capacity_mah=capacity_mah, nominal_v=3.6)

    def is_low(self) -> bool:
        return self.current_v < 3.5

    def recommend_charge_current_a(self) -> float:
        return (self.capacity_mah / 1000.0) * 0.5


class LiPo4S(Battery):
    def __init__(self, capacity_mah: int = 5200):
        super().__init__(capacity_mah=capacity_mah, nominal_v=14.8)
        self.__cells = [3.70, 3.70, 3.70, 3.70]
        self.set_voltage(sum(self.__cells))

    def is_low(self) -> bool:
        return self.current_v < 14.0

    def set_storage_mode(self) -> None:
        self.__cells = [3.85, 3.85, 3.85, 3.85]
        self.set_voltage(sum(self.__cells))    

def estimate_energy_wh(battery: Battery) -> float:
    return (battery.capacity_mah / 1000.0) * battery.nominal_v
```
3. Делаем из папок пакеты:
```python
# autopilot/__init__.py
from .machines import Motor, Machine, Rover, Copter
from .diagnostics import make_status_line, can_start
```
```python
# power/__init__.py
from .batteries import Battery, LiIon18650, LiPo4S, estimate_energy_wh
```
4. дополнительный файл с функцией для вывода информации
```python
from power import Battery, estimate_energy_wh
from .machines import Machine


def can_start(machine: Machine) -> bool:
    return machine.armed and (not machine.battery.is_low())


def make_status_line(machine: Machine) -> str:
    b: Battery = machine.battery
    energy = estimate_energy_wh(b)
    return (f"[{machine.vehicle_type.upper()}] mode={machine.mode:7} "
            f"armed={machine.armed} "
            f"battery={b.current_v:5.2f}V low={b.is_low()} "
            f"est_energy={energy:6.2f}Wh "
            f"motor_running={machine.motor.running}")
```

5. точка входа
```python
from datetime import datetime

import power 
from autopilot import Rover, Copter, Motor, make_status_line, can_start 


def main() -> None:
    now = datetime.now()
    print(f"Запуск проекта: {now:%Y-%m-%d %H:%M:%S}")

    # делаем батарейки
    liion = power.LiIon18650(capacity_mah=3500)
    lipo = power.LiPo4S(capacity_mah=5200)

    #поменять напряжение
    liion.set_voltage(3.40)  
    lipo.set_voltage(15.10) 

    # создаём БПА
    rover = Rover(battery=liion, motor=Motor(20.0), wheels=6)
    copter = Copter(battery=lipo, motor=Motor(250.0), rotors=4)

    rover.set_mode("AUTO")
    copter.set_mode("LOITER")

    rover.arm()
    copter.arm()

    rover.start()
    copter.start()

    # выводим результат в консоль
    print(f"Rover can_start={can_start(rover)} | {make_status_line(rover)}")
    print(f"Copter can_start={can_start(copter)} | {make_status_line(copter)}")
    print(f"LiIon recommended charge current: {liion.recommended_charge_current_a():.2f}A")
    print(f"LiPo estimated energy: {power.estimate_energy_wh(lipo):.2f}Wh")


if __name__ == "__main__":
    main()
```
