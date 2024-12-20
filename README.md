#!/bin/python

import sys
import numpy as np
from mpl_toolkits.mplot3d import Axes3D
from mpl_toolkits.mplot3d.art3d import Poly3DCollection
import matplotlib.pyplot as plt
import xml.etree.ElementTree as ET
import configparser
import zmq
import json

# Global variables
fig = None
ax = None
points = []
excluded_points = []
connections = []
num_of_layers = 0
layers = []
faces = []

def init_script(mesh_file):
    try:
        tree = ET.parse(mesh_file)
    except FileNotFoundError:
        raise FileNotFoundError("Mesh file not found!")
    else:
        root = tree.getroot()
        config = configparser.ConfigParser()
        config.read('/home/srijan/ratatoskr/hardware/config.ini')
        global num_of_layers
        num_of_layers = int(config['Hardware']['z'])
        proc_elemnt_ids = []
        for nodeType in root.find('nodeTypes').iter('nodeType'):
            if nodeType.find('model').attrib['value'] == 'ProcessingElement':
                proc_elemnt_ids.append(int(nodeType.attrib['id']))
        global points
        for node in root.find('nodes').iter('node'):
            if int(node.find('nodeType').attrib['value']) not in proc_elemnt_ids:
                x = float(node.find('xPos').attrib['value'])
                y = float(node.find('yPos').attrib['value'])
                z = float(node.find('zPos').attrib['value'])
                layer = int(node.find('layer').attrib['value'])
                points.append(([x, y, z], layer))
            else:
                global excluded_points
                excluded_points.append(int(node.attrib['id']))
        global connections
        for con in root.find('connections').iter('con'):
            valid_con = True
            for port in con.find('ports').iter('port'):
                if int(port.find('node').attrib['value']) in excluded_points:
                    valid_con = False
                    break
            if valid_con:
                connection = [
                    int(con.find('ports')[0].find('node').attrib['value']),
                    int(con.find('ports')[1].find('node').attrib['value'])
                ]
                connections.append(connection)

def create_fig():
    global fig
    fig = plt.figure()
    global ax
    ax = fig.add_subplot(111, projection='3d')
    ax.set_xlabel('X')
    ax.set_ylabel('Y')
    ax.set_zlabel('Z')

def vertical_horizontal_connection(p1_ix, p2_ix):
    x = [points[p1_ix][0][0], points[p2_ix][0][0]]
    y = [points[p1_ix][0][1], points[p2_ix][0][1]]
    z = [points[p1_ix][0][2], points[p2_ix][0][2]]
    ax.plot(x, y, z, color='black')

def solve_diagonal_connection(p1, p2):
    x, y, z = [], [], []
    if p1[2] > p2[2]:
        x.extend([p1[0], p1[0], p2[0]])
        y.extend([p1[1], p1[1], p2[1]])
        z.extend([p1[2], p2[2], p2[2]])
    else:
        x.extend([p2[0], p2[0], p1[0]])
        y.extend([p2[1], p2[1], p1[1]])
        z.extend([p2[2], p1[2], p1[2]])
    ax.plot(x, y, z, color='black')

def plot_connections():
    for c in connections:
        p1_ix, p2_ix = c
        p1, p2 = points[p1_ix][0], points[p2_ix][0]
        if p1[0] != p2[0] and p1[1] != p2[1]:
            solve_diagonal_connection(p1, p2)
        else:
            vertical_horizontal_connection(p1_ix, p2_ix)

def annotate_points():
    for i, p in enumerate(points):
        x, y, z = p[0]
        ax.text(x, y, z, str(i), size=12, color='red')

def colorize_nodes(colorValues):
    points_coordinates = np.array([p[0] for p in points])
    xs, ys, zs = points_coordinates[:, 0], points_coordinates[:, 1], points_coordinates[:, 2]
    global routerHeat
    routerHeat = ax.scatter(xs, ys, zs, c=colorValues, cmap='inferno', s=200)

def main():
    network_file = '/home/srijan/ratatoskr/simulator/config/network.xml'
    try:
        network_file = sys.argv[1]
    except IndexError:
        pass
    init_script(network_file)
    create_fig()
    plot_connections()
    colorize_nodes(range(len(points)))

    timeStamp = ax.text(2, 2, 2, "Time: 0.00 ns", size=12, color='red')
    averageRouterLoad = [0] * len(points)

    context = zmq.Context()
    socket = context.socket(zmq.REQ)
    socket.connect("tcp://localhost:5555")

    blink_state = True  # To toggle blinking

    for request in range(2000000):
        socket.send_string("Hello")
        message = socket.recv()

        try:
            data = json.loads(message)
        except json.JSONDecodeError:
            print("Received invalid data.")
            continue

        try:
            # Time and load updates
            time = float(data["Time"]["time"])
            currentLoad = [
                float(data["Data"][routerindex]['averagebufferusage']) if routerindex < len(data["Data"]) else 0
                for routerindex in range(len(points))
            ]

            # Update the color for router heatmap
            if 'routerHeat' in globals():
                routerHeat.remove()
            colorize_nodes(currentLoad)

            # Blinking time logic
            blink_state = not blink_state  # Toggle state
            timeStamp.remove()  # Remove the old timestamp
            color = 'red' if blink_state else 'blue'  # Alternate color for blinking effect
            timeStampValue = f"Time: {time / 1000:.2f} ns"
            timeStamp = ax.text(0, 1, 1, timeStampValue, size=12, color=color)

        except KeyError as e:
            print(f"Missing key in received data: {e}")
        except Exception as e:
            print(f"Unexpected error: {e}")

        # Ensure consistent GUI refresh
        plt.pause(1 / 30)
        fig.canvas.draw_idle()  # Explicitly trigger a redraw

    plt.show()

if _name_ == "_main_":
    main()
