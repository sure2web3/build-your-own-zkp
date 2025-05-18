# conversion-from-projective-to-affine
```python
import numpy as np
import matplotlib.pyplot as plt
from mpl_toolkits.mplot3d import Axes3D

# Example projective point (X, Y, Z)
X, Y, Z = 2, 3, 1

# Convert to affine coordinates if Z != 0
if Z != 0:
    x_affine = X / Z
    y_affine = Y / Z
else:
    raise ValueError("Z cannot be zero for affine conversion.")

fig = plt.figure(figsize=(12, 6))

# 3D plot for projective coordinates
ax1 = fig.add_subplot(121, projection='3d')
ax1.scatter(X, Y, Z, color='blue', s=100, label='Projective Point (X:Y:Z)')
ax1.set_xlabel('X')
ax1.set_ylabel('Y')
ax1.set_zlabel('Z')
ax1.set_title('Projective Point in 3D')
ax1.legend()
ax1.grid(True)

# 2D plot for affine coordinates
ax2 = fig.add_subplot(122)
ax2.scatter(x_affine, y_affine, color='red', s=100, label='Affine Point (x, y)')
ax2.set_xlabel('x')
ax2.set_ylabel('y')
ax2.set_title('Affine Point in 2D')
ax2.legend()
ax2.grid(True)

plt.suptitle('Conversion from Projective (X:Y:Z) to Affine (x, y) Coordinates')
plt.tight_layout()
plt.show()
```
# doubling-elliptic-curve
```python
import numpy as np
import matplotlib.pyplot as plt

# Elliptic curve parameters
a = 0
b = 5

# Define the elliptic curve equation
def elliptic_curve(x):
    return np.sqrt(x**3 + a * x + b), -np.sqrt(x**3 + a * x + b)

# Define the function to double a point
def double_point(x1, y1, a):
    m = (3 * x1**2 + a) / (2 * y1)
    x3 = m**2 - 2 * x1
    y3 = m * (x1 - x3) - y1
    return x3, y3

# Define the function to find the intersection of the tangent line with the curve
def find_intersection(x1, y1, a, b):
    m = (3 * x1**2 + a) / (2 * y1)
    c = y1 - m * x1
    
    # Solve the equation m*x + c = sqrt(x^3 + a*x + b)
    # This leads to the cubic equation x^3 - m^2*x^2 + (a - 2*m*c)*x + (b - c^2) = 0
    # We know x1 is a double root, so we can factor it out
    # The third root x2 can be found from the sum of roots: x1 + x1 + x2 = m^2
    x2 = m**2 - 2*x1
    y2 = m * x2 + c
    
    return x2, y2

# Generate x values for the elliptic curve
x = np.linspace(-3, 3, 400)
y_pos, y_neg = elliptic_curve(x)

# Select a point P
x1 = 1
y1 = np.sqrt(x1**3 + a * x1 + b)

# Calculate 2P
x3, y3 = double_point(x1, y1, a)

# Find the intersection of the tangent line with the curve
x2, y2 = find_intersection(x1, y1, a, b)

# Calculate the slope of the tangent line
m = (3 * x1**2 + a) / (2 * y1)

# Calculate the intercept of the tangent line
c = y1 - m * x1

# Generate x values for the tangent line
x_tangent = np.linspace(x1 - 2, x1 + 2, 200)
y_tangent = m * x_tangent + c

# Create the plot
plt.figure(figsize=(10, 8))

# Plot the elliptic curve
plt.plot(x, y_pos, 'b', label=r'$y = \sqrt{x^3 + ax + b}$')
plt.plot(x, y_neg, 'b')

# Plot point P
plt.scatter(x1, y1, color='red', s=80, label='Point P')

# Plot the tangent line
plt.plot(x_tangent, y_tangent, 'g--', label='Tangent line at P')

# Plot the intersection point
plt.scatter(x2, y2, color='purple', s=80, label='Intersection point')

# Plot point 2P
plt.scatter(x3, y3, color='orange', s=80, label='Point 2P')

# Draw a dashed line between intersection point and 2P
plt.plot([x2, x3], [y2, -y2], 'r--', label='Reflection line')

# Set plot properties
plt.title('Elliptic Curve Point Doubling Illustration', fontsize=16)
plt.xlabel('x', fontsize=14)
plt.ylabel('y', fontsize=14)
plt.grid(True, linestyle='--', alpha=0.7)
plt.legend(fontsize=12)
plt.axis('equal')  # Make axes equal for better visualization
plt.ylim(-5, 5)    # Adjust y-axis limits

# Show the plot
plt.tight_layout()
plt.show()
```
# mapping-integers-to-field-elements
```python
import numpy as np
import matplotlib.pyplot as plt

# Set modulus p for the finite field
p = 7

# Select a set of integers n, including values not divisible by p
n_values = np.array([0, 1, 2, 3, 5, 6, 7, 8, 10, 13, 14, 15, 20])
field_elements = n_values % p

fig = plt.figure(figsize=(10, 6))
ax = fig.add_subplot(111, projection='3d')

# Plot each integer as a point in (n, p, r) space
sc = ax.scatter(n_values, [p]*len(n_values), field_elements, c=field_elements, cmap='viridis', s=80)

# Annotate each point
for n, r in zip(n_values, field_elements):
    ax.text(n, p, r, f"n={n}\nr={r}", fontsize=9, ha='center', va='bottom')

ax.set_xlabel('Integer n')
ax.set_ylabel('Modulus p')
ax.set_zlabel('Field Element r = n mod p')
ax.set_title(f'Mapping Integers to Field Elements in Finite Field $\\mathbb{{Z}}_p$ (p={p})')

plt.colorbar(sc, label='Field Element Value')
plt.tight_layout()
plt.show()
```