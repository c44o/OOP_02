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

    def estimate_energy_mh(battery: Battery) -> float:
        return (battery.capacity_mah / 1000.0) * battery.nominal_v
```
