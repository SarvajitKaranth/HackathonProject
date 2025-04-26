from scipy.integrate import quad
import subprocess
import re
import pandas as pd
import math
#for calculating air density and kinematic viscocity
temp = 273.15 + int(input('what is the air temperature in Celcius?: '))
pressure = int(input('what is the air  pressure in Pa?: '))
airDensity = pressure / (287.58*temp)
visc = (1.789e-5) * (((temp/288.15)**1.5) * (398.55/(110.4+temp)))
Howmany = int(input('how many NACA airfoils would you like to compare? '))
r = 1
airfoilList = []
while r <= Howmany:
    print('#', r)
    airfoil = str(input('Naca Airfoil #: '))
    airfoilList.append(airfoil)
    r += 1 
print(airfoilList)
# --- XFOIL Management ---
xfoil_cache = {}

def parse_xfoil_output(output):
    cl_cd_pattern = re.compile(r'CL\s*=\s*([-+]?[0-9]*\.?[0-9]+)\s+.*CD\s*=\s*([-+]?[0-9]*\.?[0-9]+)')
    match = cl_cd_pattern.search(output)
    if match:
        cl = float(match.group(1))
        cd = float(match.group(2))
        return cl, cd
    else:
        raise ValueError("CL and CD values not found in the output.")

def run_xfoil(airfoil_name: str, Re: float):
    Re = int(round(Re, -3))  # round to nearest 1000 for caching


    if (airfoil_name, Re) in xfoil_cache:
        return xfoil_cache[(airfoil_name, Re)]

    process = subprocess.Popen(
        [r"XFOIL6.99\xfoil.exe"],
        stdin=subprocess.PIPE,
        stdout=subprocess.PIPE,
        stderr=subprocess.PIPE,
        text=True
    )

    commands = f"""
NACA {airfoil_name}
OPER
VISC {Re}
ALFA 9

"""
    output, error = process.communicate(commands)

    if error:
        print("XFOIL:", error)

    cl, cd = parse_xfoil_output(output)
    xfoil_cache[(airfoil_name, Re)] = (cl, cd)
    return cl, cd

# --- Parameters ---
slen = [600,0,500,700,0,400,200,800,0,0,1000,0]
trad = [16,16,327,18,18,50,45,30,40,50,80,300]
tlen = [70,70,320,75,75,90,85,100,100,100,120,120]

pwr = 782984.9 # Watts
mass = 728     # kg
fuelKginit = int(input('how much fuel do you want to run for the lap? (max 110kg, min 2kg): '))
fuelKg = fuelKginit
Wet = int(input('Wet or Dry? (if wet, enter "1", if dry, enter "2"): '))
tireType = int(input('soft, medium, or hard tires? (if soft, enter 1. if medium, enter 2. if hard, enter 3.): '))
if tireType == 1:
    cFric = 0.7
elif tireType ==2:
    cFric = 1.0
else:
    cFric = 1.5
if Wet == 1:
    cFric = cFric - 0.3
#airDensity = 1.184
decel = 49.05  # m/s²
A = 0.696      # m²
#visc = 1.73e-5 # m²/s
deltaT = float(input('input time step - this will decide accuracy and how long code will take to run: '))  # time step (s)
fuel_rate_per_meter = 0.0003465  # kg/m

def ReGen(FS):
    return (airDensity * FS * 1.5) / visc

def turnT(dist, rad, airfoil):
    global fuelKg

    totalM = mass + fuelKg
    vmaxInit = math.sqrt((totalM * cFric * 9.81 * rad) / (totalM - (0.5 * airDensity * 1.87 * A * rad)))
    vL = vmaxInit
    x = 0
    t = 0
    while x <= dist:
        cl, cd = run_xfoil(airfoil, ReGen(vL))
        deltaV = math.sqrt((totalM * cFric * 9.81 * rad) / (totalM - (0.5 * airDensity * cd * A * rad)))
        dx = deltaV * deltaT
        x += dx
        t += deltaT
        vL = deltaV

        fuelKg -= fuel_rate_per_meter * dx
        totalM = mass + fuelKg

        print(f"Turn: x={x:.2f}, t={t:.2f}, v={vL:.2f}, cl={cl:.3f}, cd={cd:.5f}, fuel={fuelKg:.2f}")

    return t, vL

def straight(maxX, vlast, vmax, airfoilname):
    global fuelKg

    v = vlast
    x = 0
    t = 0
    while x <= maxX:
        totalM = mass + fuelKg
        if v == 0:
            hp = pwr
            reN = 1000
        else:
            hp = pwr / v
            reN = ReGen(v)

        cl, cd = run_xfoil(airfoilname, reN)
        Fd = 0.5 * airDensity * v**2 * cd * A
        Fl = 0.5 * airDensity * v**2 * cl * A
        Ff = cFric * ((totalM * 9.81) + Fl)
        a = (hp - (Fd + Ff)) / totalM

        v += a * deltaT
        dx = v * deltaT
        x += dx
        t += deltaT

        fuelKg -= fuel_rate_per_meter * dx

        print(f"Straight: x={x:.2f}, v={v:.2f}, a={a:.3f}, t={t:.2f}, fuel={fuelKg:.2f}")

    # Deceleration check
    dtime = 0
    if vmax**2 < v**2:
        dV = v - vmax
        dtime = dV / decel
        print(f"DECELERATING: v={v:.2f} to vmax={vmax:.2f}, dtime={dtime:.2f}")

    return t + dtime

# --- Main Simulation ---
totalT = 0
vLList = [0]

# Turns
#for i in range(len(trad)):
  #  t, vL = turnT(tlen[i], trad[i])
  #  totalT += t
   # vLList.append(vL)

#print("\nTurn Exit Speeds:", vLList)

# Straights
#for i in range(len(slen)):
   # vlast = vLList[i]
    #vmax = vLList[i+1]
    #maxX = slen[i]
    #time = straight(maxX, vlast, vmax)
   # totalT += time
timeList = []
fuellist =[]
for i in range(len(airfoilList)):
    totalT = 0
    vLlist = [0]
    t=0
    vL=0
    totalT = 0
    vlast = 0
    vmax = 0
    maxX = 0
    time = 0
    pwr = 782984.9 # Watts
    mass = 728     # kg
    fuelKg = fuelKginit
    #airDensity = 1.184
    decel = 49.05  # m/s²
    A = 0.696      # m²
    #visc = 1.73e-5 # m²/s
    for k in range(len(trad)):
        t, vL = turnT(tlen[k], trad[k],airfoilList[i])
        totalT += t
        vLList.append(vL)
    print('\nTurn Exit Speeds:',vLList)
    for p in range(len(slen)):
        vlast = vLList[p]
        vmax = vLList[p+1]
        maxX = slen[p]
        time = straight(maxX, vlast, vmax, airfoilList[i])
        totalT += time
    timeList.append(totalT)
    fuellist.append(fuelKg)
for i in range(len(airfoilList)):
    percentleft = ((fuellist[i])/fuelKginit) * 100
    print('time(s) for',  airfoilList[i], 'is', timeList[i], "with", percentleft, "% fuel left")

#print("\nTotal Lap Time: (s)", totalT)
#print("Final Fuel Left (kg):", fuelKg)
#percentleft = (fuelKg/fuelKginit) * 100
#print("after one lap", percentleft, "% fuel left")
