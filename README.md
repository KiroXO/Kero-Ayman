#include <GL/gl.h>
#include <GL/glu.h>
#include <GL/freeglut.h>
#include <time.h>
#include <cmath>
#include <iostream>

// Window dimensions
int width = 800, height = 600;


float playerPosX = 0.0f;    // X-axis position (centered)
float playerPosZ = -30.0f;  // Z-axis position (center of the goal)

// Ball position and movement variables
float ballPos[3] = {0.0f, 1.0f, 0.0f}; // Ball position (x, y, z)
float ballVelocity[3] = {0.0f, 0.0f, 0.0f}; // Ball velocity (dx, dy, dz)

// Color schemes for Dark and Light mode
bool isDarkMode = true;  // By default, start with dark mode
float backgroundColor[3] = {0.0f, 0.0431f, 0.3451f}; // Dark mode background color
float objectColor[3] = {1.0f, 1.0f, 1.0f};    // Light color for objects in dark mode


// Camera position and look-at parameters
float cameraPos[3] = {0.0f, 5.0f, 10.0f};  // Camera position
float lookAt[3] = {0.0f, 0.0f, 0.0f};     // Camera look-at point
float cameraUp[3] = {0.0f, 1.0f, 0.0f};   // Camera up vector


// Camera rotation angle (in degrees)
float angle = 0.0f;

// Ball properties
float ballRadius = 1.0f;
float ballPosX = 0.0f, ballPosY = ballRadius, ballPosZ = 0.0f;
float ballVelocityX = 0.0f, ballVelocityY = 0.0f, ballVelocityZ = 0.0f;
float gravity = -0.098f;

// Variables for input
int isKicking = 0;
int hasKicked = 0; // Flag to ensure the ball is kicked only once
clock_t kickStartTime;
float kickForce = 0.0f;
clock_t kickTime; // Time of the last kick



// Function to set up the camera view
void setupCamera() // Function to compute the camera's position based on the angle of rotation around the Y-axis
{
    float radius = 10.0f; // Radius of the camera's orbit around the Y-axis

    // Calculate the new camera position using trigonometry
    cameraPos[0] = radius * sin(angle * M_PI / 180.0f);  // X position: radius * sin(angle)
    cameraPos[2] = radius * cos(angle * M_PI / 180.0f);  // Z position: radius * cos(angle)
}



// Function to update physics (ball movement)
void updatePhysics() {
    // Apply gravity to the vertical velocity (Y-axis)
    ballVelocityY += gravity;

    // Update ball position based on velocity
    ballPos[0] += ballVelocityX;  // X position changes with X velocity
    ballPos[1] += ballVelocityY;  // Y position changes with Y velocity
    ballPos[2] += ballVelocityZ;  // Z position changes with Z velocity

    // If the ball hits the ground (Y <= ballRadius), stop its vertical velocity and reset vertical position
    if (ballPos[1] <= ballRadius) {
        ballPos[1] = ballRadius;   // Ensure the ball stays on the ground
        ballVelocityY = 0.0f;      // Reset the Y velocity
    }
}

// Function to reset the ball position after a certain amount of time or when out of bounds
void resetBallIfNeeded() {
    // Check if the ball is outside the pitch boundaries
    if (ballPos[0] < -30.0f || ballPos[0] > 30.0f || ballPos[2] < -20.5f || ballPos[2] > 20.5f) {
        // Reset the ball's position to the center
        ballPos[0] = 0.0f;
        ballPos[1] = ballRadius; // Keep the ball at ground level
        ballPos[2] = 0.0f;

        // Reset the ball's velocity to stop movement
        ballVelocityX = 0.0f;
        ballVelocityY = 0.0f;
        ballVelocityZ = 0.0f;

        // Allow the ball to be kicked again
        hasKicked = 0;
    }
}


//Inputs (Keyboard) >>>>>>>>>>>>>>>>>>>>>>>>
void specialKeys(int key, int x, int y) {
    float rotationSpeed = 2.0f; // Speed of camera rotation (degrees per key press)

    if (key == GLUT_KEY_RIGHT) { // Rotate camera counterclockwise
        angle -= rotationSpeed; // Decrease the angle to rotate counterclockwise
    }
    else if (key == GLUT_KEY_LEFT) { // Rotate camera clockwise
        angle += rotationSpeed; // Increase the angle to rotate clockwise
    }

    // Keep the angle within 0 to 360 degrees
    if (angle >= 360.0f) angle -= 360.0f;
    if (angle < 0.0f) angle += 360.0f;

    glutPostRedisplay(); // Request a redraw of the scene
}


// Keyboard input handling
void keyboard(unsigned char key, int x, int y)
{

    if (key == 'k' && !hasKicked)
        {  // Spacebar key, only if the ball hasn't been kicked yet
        if (!isKicking)
        {
            // Start timing when the spacebar is pressed down
            isKicking = 1;
            kickStartTime = clock();
        }
    }
     if (key == 'a' || key == 'A') {
        // Move left
        playerPosZ += 0.5f;
    }
    if (key == 'd' || key == 'D') {
        // Move right
        playerPosZ -= 0.5f;
    }
}

void keyboardUp(unsigned char key, int x, int y) {
    if (key == 'k' && !hasKicked) {  // Spacebar key, only if the ball hasn't been kicked yet
        if (isKicking) {
            // Calculate the force based on how long the spacebar was held down
            kickForce = (float)(clock() - kickStartTime) / CLOCKS_PER_SEC;
            kickForce = fmin(kickForce, 10.0f); // Limit max kick force to 10.0

            // Calculate the direction of the camera
            float directionX = sin(angle * M_PI / 180.0f);
            float directionZ = -cos(angle * M_PI / 180.0f);

            // Apply the force to the ball in the direction the camera is looking
            ballVelocityX = kickForce * directionX;  // Apply force in the X direction
            ballVelocityZ = kickForce * directionZ;  // Apply force in the Z direction

            // After the spacebar is released, mark the ball as kicked
            isKicking = 0;
            hasKicked = 1; // Prevent further velocity changes
            kickTime = clock(); // Mark the time when the ball was kicked
        }
    }
}




//Everything We have drawn in the scene
// Function to create a cylinder with a given radius and height
void drawCylinder(float radius, float height, int slices, int stacks) {
    GLUquadric* quad = gluNewQuadric();
    gluCylinder(quad, radius, radius, height, slices, stacks);
    gluDeleteQuadric(quad);
}

// Function to draw a hollow pipe for the GOAL POSTS
void drawHollowPipe(float outerRadius, float height) {
    // Draw the outer cylinder (the main body of the pipe)
    glPushMatrix();
    glColor3f(1.0f, 1.0f, 1.0f); // White for outer surface
    drawCylinder(outerRadius, height, 10, 10);
    glPopMatrix();
}


void DrawCircle(float x, float z, double radius, int a)
{
    int i;
    double angle;
    glPushMatrix();
    glRotatef(90.0f, 1.0f, 0.0f, 0.0f);
    glBegin(GL_LINE_STRIP);
    glColor3f(1.0, 1.0, 1.0);

    for(i=0; i<=a; i++)
    {
        angle = i*3.14 / 180;
        glVertex2f(radius* cos(angle) + x, radius*sin(angle) + z);
    }


    glEnd();

    glPopMatrix();
}


// Function to draw a cube
void drawCube(float x, float y, float z, float width, float height, float depth) {
    glPushMatrix();
    glTranslatef(x, y, z); // Move the cube to (x, y, z)
    glScalef(width, height, depth); // Scale the cube

    glBegin(GL_QUADS); // Begin drawing the 6 faces of the cube

    // Front face (Red)
    glColor3f(1.0, 0.0, 0.0);
    glVertex3f(-0.5, -0.5, 0.5);
    glVertex3f(0.5, -0.5, 0.5);
    glVertex3f(0.5, 0.5, 0.5);
    glVertex3f(-0.5, 0.5, 0.5);

    // Back face (Green)
    glColor3f(0.0, 1.0, 0.0);
    glVertex3f(-0.5, -0.5, -0.5);
    glVertex3f(-0.5, 0.5, -0.5);
    glVertex3f(0.5, 0.5, -0.5);
    glVertex3f(0.5, -0.5, -0.5);

    // Left face (Blue)
    glColor3f(0.0, 0.0, 1.0);
    glVertex3f(-0.5, -0.5, -0.5);
    glVertex3f(-0.5, -0.5, 0.5);
    glVertex3f(-0.5, 0.5, 0.5);
    glVertex3f(-0.5, 0.5, -0.5);

    // Right face (Yellow)
    glColor3f(1.0, 1.0, 0.0);
    glVertex3f(0.5, -0.5, -0.5);
    glVertex3f(0.5, 0.5, -0.5);
    glVertex3f(0.5, 0.5, 0.5);
    glVertex3f(0.5, -0.5, 0.5);

    // Top face (Purple)
    glColor3f(1.0, 0.0, 1.0);
    glVertex3f(-0.5, 0.5, -0.5);
    glVertex3f(0.5, 0.5, -0.5);
    glVertex3f(0.5, 0.5, 0.5);
    glVertex3f(-0.5, 0.5, 0.5);

    // Bottom face (Cyan)
    glColor3f(0.0, 1.0, 1.0);
    glVertex3f(-0.5, -0.5, -0.5);
    glVertex3f(-0.5, -0.5, 0.5);
    glVertex3f(0.5, -0.5, 0.5);
    glVertex3f(0.5, -0.5, -0.5);

    glEnd(); // End drawing quads
    glPopMatrix();
}


// Function to draw a half-circle at a specific position (radius, angle, and number of segments)
void drawLeftHalfCircle(float x, float z, float radius, float angleStart, float angleEnd) {
    int segments = 20;  // Number of segments for the half-circle (higher means smoother curve)
    glPushMatrix();
    glTranslatef(x, 0.0f, z);  // Translate to the position of the circle
    glRotatef(-90.0f, 0.0f, 1.0f, 0.0f);
    glBegin(GL_LINE_STRIP);  // Draw the arc using a line strip (can change to GL_POLYGON for a filled circle)
    for (int i = 0; i <= segments; ++i) {
        float angle = angleStart + (angleEnd - angleStart) * (i / (float)segments); // Interpolate angles
        float xPos = radius * cos(angle);  // Calculate X position on the circle
        float zPos = radius * sin(angle);  // Calculate Z position on the circle
        glVertex3f(xPos, 0.0f, zPos);  // Draw vertex
    }
    glEnd();
    glPopMatrix();
}


// Function to draw a half-circle at a specific position (radius, angle, and number of segments)
void drawRightHalfCircle(float x, float z, float radius, float angleStart, float angleEnd) {
    int segments = 20;  // Number of segments for the half-circle (higher means smoother curve)
    glPushMatrix();
    glTranslatef(x, 0.0f, z);  // Translate to the position of the circle
    glRotatef(90.0f, 0.0f, 1.0f, 0.0f);
    glBegin(GL_LINE_STRIP);  // Draw the arc using a line strip (can change to GL_POLYGON for a filled circle)
    for (int i = 0; i <= segments; ++i) {
        float angle = angleStart + (angleEnd - angleStart) * (i / (float)segments); // Interpolate angles
        float xPos = radius * cos(angle);  // Calculate X position on the circle
        float zPos = radius * sin(angle);  // Calculate Z position on the circle
        glVertex3f(xPos, 0.0f, zPos);  // Draw vertex
    }
    glEnd();
    glPopMatrix();
}


// Function to draw the football pitch
void drawPitch() {
    glColor3f(0.0f, 1.0f, 0.0f); // Green for the pitch
    glPushMatrix();
    glBegin(GL_QUADS);
    glVertex3f(-30.0f, 0.0f, -20.5f); // Bottom-left corner
    glVertex3f( 30.0f, 0.0f, -20.5f); // Bottom-right corner
    glVertex3f( 30.0f, 0.0f,  20.5f); // Top-right corner
    glVertex3f(-30.0f, 0.0f,  20.5f); // Top-left corner
    glEnd();




    glPushMatrix();
    glLineWidth(10.0);
    glColor3f(1.0f, 1.0f, 1.0f); // Red color for the Z-axis line
    glBegin(GL_LINES);  // Draw a line
    glVertex3f(0.0f, 0.0f, -20.5f);  // Start of the Z-axis line
    glVertex3f(0.0f, 0.0f, 20.5f);   // End of the Z-axis line
    glEnd();
    glPopMatrix();


    DrawCircle(0,0, 2.0,360);

    glPopMatrix();
}

// Function to draw a football
void drawFootball(float x, float y, float z) {
    glPushMatrix();
    glTranslatef(-x, y, z);
    glColor3f(1.0f, 1.0f, 1.0f); // White ball
    glutSolidSphere(0.5f, 20, 10); // Draw a solid sphere (football)
    glPopMatrix();
}


// Function to draw goals
void drawGoal(float x, float z) {
    glPushMatrix();
    glColor3f(1.0f, 1.0f, 1.0f); // White goalposts
    glTranslatef(x, 3.0f, z);

    // Draw the vertical goalposts (4 posts in total)

    // Rotate the crossbar 90 degrees around the X-axis
    glPushMatrix();
    glRotatef(-90.0f, 1.0f, 0.0f, 0.0f);

    glPushMatrix();
    glTranslatef(-15.0f, -5.0f, -3.0f); // Left post
    //glScalef(0.2f, 6.0f, 0.2f);
    drawHollowPipe(0.2f, 6.0f);
    glPopMatrix();


    // Rotate the crossbar 90 degrees around the X-axis
    glPushMatrix();
    glRotatef(90.0f, 1.0f, 0.0f, 0.0f);

    glPushMatrix();
    glTranslatef(-15.0f, 20.0f, -3.0f); // Right post
    //glScalef(0.2f, 6.0f, 0.2f);
    drawHollowPipe(0.2f, 6.0f);
    glPopMatrix();

    // Rotate the crossbar 90 degrees around the Z-axis
    //glPushMatrix();
    //glRotatef(90.0f, 0.0f, 1.0f, 0.0f);

    // Draw the crossbar
    glPushMatrix();
    glTranslatef(-14.95f, 3.0f, 5.0f);
    //glScalef(10.0f, 0.2f, 0.2f); // Scale to make the crossbar
    drawHollowPipe(0.2f, 15.0f); // Crossbar
    glPopMatrix();


    glPopMatrix();

}



// Modify the function to draw the penalty area to include the half-circle
void drawLeftPenaltyArea(){
    glPushMatrix();
    glColor3f(1.0f, 1.0f, 1.0f); // White for the penalty area borders
    glLineWidth(5.0);

    // Draw the outer penalty area borders
    glBegin(GL_LINES);
    glVertex3f( -30.0f, 0.0f,  13.0f); // Top-left corner
    glVertex3f( -15.0f, 0.0f,  13.0f); // Top-right corner
    glVertex3f( -15.0f, 0.0f,  13.0f); // Top-right corner
    glVertex3f( -15.0f, 0.0f,  -13.0f); // Bottom-right corner
    glVertex3f( -15.0f, 0.0f,  -13.0f); // Bottom-right corner
    glVertex3f( -30.0f, 0.0f,  -13.0f); // Bottom-left corner
    glVertex3f( -30.0f, 0.0f,  -13.0f); // Bottom-left corner
    glVertex3f( -30.0f, 0.0f,  13.0f); // Top-left corner
    glEnd();

    // Draw the inner penalty area borders
    glBegin(GL_LINES);
    glVertex3f( -30.0f, 0.0f,  7.5f);  // Top-left corner
    glVertex3f( -20.0f, 0.0f,  7.5f);  // Top-right corner
    glVertex3f( -20.0f, 0.0f,  7.5f);  // Top-right corner
    glVertex3f( -20.0f, 0.0f,  -7.5f); // Bottom-right corner
    glVertex3f( -20.0f, 0.0f,  -7.5f); // Bottom-right corner
    glVertex3f( -30.0f, 0.0f,  -7.5f); // Bottom-left corner
    glVertex3f( -30.0f, 0.0f,  -7.5f); // Bottom-left corner
    glVertex3f( -30.0f, 0.0f,  7.5f);  // Top-left corner
    glEnd();

    // Draw the half-circle at the border of penalty area
    drawLeftHalfCircle(-15.0f, 0.0f, 5.0f, M_PI, 2 * M_PI); // Half-circle at the top-left corner, adjust radius if needed

    glPopMatrix();
}


void drawRightPenaltyArea()
{

    glPushMatrix();
    glColor3f(1.0f, 1.0f, 1.0f); // White for the penalty area borders
    glLineWidth(5.0);

    // Draw the outer penalty area borders
    glBegin(GL_LINES);
    glVertex3f( 30.0f, 0.0f,  13.0f); // Top-left corner
    glVertex3f( 15.0f, 0.0f,  13.0f); // Top-right corner
    glVertex3f( 15.0f, 0.0f,  13.0f); // Top-right corner
    glVertex3f( 15.0f, 0.0f,  -13.0f); // Bottom-right corner
    glVertex3f( 15.0f, 0.0f,  -13.0f); // Bottom-right corner
    glVertex3f( 30.0f, 0.0f,  -13.0f); // Bottom-left corner
    glVertex3f( 30.0f, 0.0f,  -13.0f); // Bottom-left corner
    glVertex3f( 30.0f, 0.0f,  13.0f); // Top-left corner
    glEnd();

    // Draw the inner penalty area borders
    glBegin(GL_LINES);
    glVertex3f( 30.0f, 0.0f,  7.5f);  // Top-left corner
    glVertex3f( 20.0f, 0.0f,  7.5f);  // Top-right corner
    glVertex3f( 20.0f, 0.0f,  7.5f);  // Top-right corner
    glVertex3f( 20.0f, 0.0f,  -7.5f); // Bottom-right corner
    glVertex3f( 20.0f, 0.0f,  -7.5f); // Bottom-right corner
    glVertex3f( 30.0f, 0.0f,  -7.5f); // Bottom-left corner
    glVertex3f( 30.0f, 0.0f,  -7.5f); // Bottom-left corner
    glVertex3f( 30.0f, 0.0f,  7.5f);  // Top-left corner
    glEnd();

    // Draw the half-circle at the border of penalty area
    drawRightHalfCircle(15.0f, 0.0f, 5.0f, M_PI, 2 * M_PI); // Half-circle at the top-left corner, adjust radius if needed

    glPopMatrix();
}


// Function to draw a pyramid
void drawPyramid(float x, float y, float z, float width, float height, float r, float g, float b) {
    glPushMatrix();
    glTranslatef(x, y, z);

    glBegin(GL_TRIANGLES);
    // Base of the pyramid (square)
    glColor3f(r * 0.7, g * 0.7, b * 0.7);
    glVertex3f(-width/2, 0, -width/2);
    glVertex3f(width/2, 0, -width/2);
    glVertex3f(width/2, 0, width/2);
    glVertex3f(-width/2, 0, width/2);
    glVertex3f(-width/2, 0, -width/2);
    glVertex3f(width/2, 0, width/2);

    // Sides of the pyramid
    glColor3f(r, g, b);
    // Front face
    glVertex3f(0, height, 0);
    glVertex3f(-width/2, 0, -width/2);
    glVertex3f(width/2, 0, -width/2);

    // Right face
    glVertex3f(0, height, 0);
    glVertex3f(width/2, 0, -width/2);
    glVertex3f(width/2, 0, width/2);

    // Back face
    glVertex3f(0, height, 0);
    glVertex3f(width/2, 0, width/2);
    glVertex3f(-width/2, 0, width/2);

    // Left face
    glVertex3f(0, height, 0);
    glVertex3f(-width/2, 0, width/2);
    glVertex3f(-width/2, 0, -width/2);

    glEnd();
    glPopMatrix();
}


// Function to draw a window
void drawWindow(float x, float y, float z, float width, float height, float depth) {
    glPushMatrix();
    glTranslatef(x, y, z); // Move window to (x, y, z)
    glScalef(width, height, depth); // Scale window

    glBegin(GL_QUADS); // Drawing the window as a flat plane

    // Front face of the window (Light blue)
    glColor3f(1.0f, 1.0f, 1.0f);
    glVertex3f(-0.5, -0.5, 0.01);
    glVertex3f(0.5, -0.5, 0.01);
    glVertex3f(0.5, 0.5, 0.01);
    glVertex3f(-0.5, 0.5, 0.01);

    glEnd();
    glPopMatrix();
}

//Buildings Function
void drawBuilding() {

    // Draw The Main Building (Middle) ..
    // Draw the base (Main building part)
    drawCube(0.0f, 8.0f, -28.0f, 8.0f, 16.0f, 8.0f); // Base dimensions: 2x4x2

    // Draw windows on the front wall
    drawWindow(-2.0f, 14.0f, -23.9f, 1.5f, 1.8f, 0.0f); // Left window
    drawWindow(2.0f, 14.0f, -23.9f, 1.5f, 1.8f, 0.0f);  // Right window
    drawWindow(-2.0f, 11.0f, -23.9f, 1.5f, 1.8f, 0.0f); // Left window
    drawWindow(2.0f, 11.0f, -23.9f, 1.5f, 1.8f, 0.0f);  // Right window
    drawWindow(-2.0f, 8.0f, -23.9f, 1.5f, 1.8f, 0.0f); // Left window
    drawWindow(2.0f, 8.0f, -23.9f, 1.5f, 1.8f, 0.0f);  // Right window
    drawWindow(0.0f, 2.0f, -23.9f, 2.0f, 4.0f, 0.0f);  // Door

    // Draw windows on the back wall
    drawWindow(-2.0f, 14.0f, -32.1f, 1.5f, 1.8f, 0.0f); // Left window
    drawWindow(2.0f, 14.0f, -32.1f, 1.5f, 1.8f, 0.0f);  // Right window
    drawWindow(-2.0f, 11.0f, -32.1f, 1.5f, 1.8f, 0.0f); // Left window
    drawWindow(2.0f, 11.0f, -32.1f, 1.5f, 1.8f, 0.0f);  // Right window
    drawWindow(-2.0f, 8.0f, -32.1f, 1.5f, 1.8f, 0.0f); // Left window
    drawWindow(2.0f, 8.0f, -32.1f, 1.5f, 1.8f, 0.0f);  // Right window


    //Draw The Building on the Right Side
    drawCube(15.0f, 8.0f, -28.0f, 8.0f, 16.0f, 8.0f);

    drawWindow(17.0f, 14.0f, -23.9f, 1.5f, 1.8f, 0.0f); // Left window
    drawWindow(13.0f, 14.0f, -23.9f, 1.5f, 1.8f, 0.0f);  // Right window
    drawWindow(17.0f, 11.0f, -23.9f, 1.5f, 1.8f, 0.0f); // Left window
    drawWindow(13.0f, 11.0f, -23.9f, 1.5f, 1.8f, 0.0f);  // Right window
    drawWindow(17.0f, 8.0f, -23.9f, 1.5f, 1.8f, 0.0f); // Left window
    drawWindow(13.0f, 8.0f, -23.9f, 1.5f, 1.8f, 0.0f);  // Right window
    drawWindow(15.0f, 2.0f, -23.9f, 2.0f, 4.0f, 0.0f);  // Door

    // Draw windows on the back wall
    drawWindow(17.0f, 14.0f, -32.1f, 1.5f, 1.8f, 0.0f); // Left window
    drawWindow(13.0f, 14.0f, -32.1f, 1.5f, 1.8f, 0.0f);  // Right window
    drawWindow(17.0f, 11.0f, -32.1f, 1.5f, 1.8f, 0.0f); // Left window
    drawWindow(13.0f, 11.0f, -32.1f, 1.5f, 1.8f, 0.0f);  // Right window
    drawWindow(17.0f, 8.0f, -32.1f, 1.5f, 1.8f, 0.0f); // Left window
    drawWindow(13.0f, 8.0f, -32.1f, 1.5f, 1.8f, 0.0f);  // Right window





    //Draw the Building on the Left Side
    drawCube(-15.0f, 8.0f, -28.0f, 8.0f, 16.0f, 8.0f);

     // Draw windows on the front wall
    drawWindow(-17.0f, 14.0f, -23.9f, 1.5f, 1.8f, 0.0f); // Left window
    drawWindow(-13.0f, 14.0f, -23.9f, 1.5f, 1.8f, 0.0f);  // Right window
    drawWindow(-17.0f, 11.0f, -23.9f, 1.5f, 1.8f, 0.0f); // Left window
    drawWindow(-13.0f, 11.0f, -23.9f, 1.5f, 1.8f, 0.0f);  // Right window
    drawWindow(-17.0f, 8.0f, -23.9f, 1.5f, 1.8f, 0.0f); // Left window
    drawWindow(-13.0f, 8.0f, -23.9f, 1.5f, 1.8f, 0.0f);  // Right window
    drawWindow(-15.0f, 2.0f, -23.9f, 2.0f, 4.0f, 0.0f);  // Door

    // Draw windows on the back wall
    drawWindow(-17.0f, 14.0f, -32.1f, 1.5f, 1.8f, 0.0f); // Left window
    drawWindow(-13.0f, 14.0f, -32.1f, 1.5f, 1.8f, 0.0f);  // Right window
    drawWindow(-17.0f, 11.0f, -32.1f, 1.5f, 1.8f, 0.0f); // Left window
    drawWindow(-13.0f, 11.0f, -32.1f, 1.5f, 1.8f, 0.0f);  // Right window
    drawWindow(-17.0f, 8.0f, -32.1f, 1.5f, 1.8f, 0.0f); // Left window
    drawWindow(-13.0f, 8.0f, -32.1f, 1.5f, 1.8f, 0.0f);  // Right window








}


// Function to draw a tree using pyramids
void drawTree(float x, float z) {
    // Tree trunk (brown color)
    drawPyramid(x, 0, z, 0.3, 2, 0.6, 0.4, 0.2);

    // Tree layers (green color)
    drawPyramid(x, 2, z, 1.5, 2, 0.2, 0.7, 0.3);
    drawPyramid(x, 3.5, z, 1.2, 1.5, 0.2, 0.8, 0.3);
    drawPyramid(x, 4.7, z, 0.8, 1, 0.2, 0.9, 0.3);
}





// Ground (grey)
void drawRGround() {
    glColor3f(0.5, 0.5, 0.5); // Set color to grey
    glBegin(GL_QUADS);
    glVertex3f(-30, 0, -36);
    glVertex3f(30, 0, -36);
    glVertex3f(30, 0, -20);
    glVertex3f(-30, 0, -20);
    glEnd();
}


// Ground (grey)
void drawLGround() {
    glColor3f(0.5, 0.5, 0.5); // Set color to grey
    glBegin(GL_QUADS);
    glVertex3f(-30, 0, 36);
    glVertex3f(30, 0, 36);
    glVertex3f(30, 0, 20);
    glVertex3f(-30, 0, 20);
    glEnd();
}



// Make sure the background is set to the correct color in the 'display' function
void display() {
    glClearColor(backgroundColor[0], backgroundColor[1], backgroundColor[2], 1.0f); // Set background color
    glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT); // Clear screen and depth buffer
    glLoadIdentity();

    setupCamera();  // Set up the camera

    // Set the camera position dynamically based on the angle
    gluLookAt(cameraPos[0], 1.0f, cameraPos[2],    // Camera at (cameraPos[0], 1.0f, cameraPos[2])
              0.0f, 0.0f, 0.0f,    // Always look at the origin (0, 0, 0)
              0.0f, 1.0f, 0.0f);   // Up direction is along the Y-axis

    drawPitch();      // Draw the football pitch
    drawFootball(ballPos[0], 0.5, ballPos[2]); // Draw the football at the center

    drawGoal(-15.0f, -12.5f); // Draw LEFT goal

    drawGoal(45.0f, -12.5f); // Draw Right goal

    drawRightPenaltyArea();
    drawLeftPenaltyArea();

    // Draw ground
    drawRGround();

    // Draw ground
    drawLGround();

    // Draw the building
    drawBuilding();

    // Draw multiple trees
    drawTree(-20, 27);
    // Draw multiple trees
    drawTree(-15, 28);
    // Draw multiple trees
    drawTree(-11, 29);
    // Draw multiple trees
    drawTree(-9, 30);

    // Draw multiple trees
    drawTree(-5, 27);
    // Draw multiple trees
    drawTree(-3, 28);
    // Draw multiple trees
    drawTree(-1, 29);
    // Draw multiple trees
    drawTree(1, 30);

    // Draw multiple trees
    drawTree(0, 27);
    // Draw multiple trees
    drawTree(2, 28);
    // Draw multiple trees
    drawTree(5, 29);
    // Draw multiple trees
    drawTree(8, 30);

    // Draw multiple trees
    drawTree(8, 27);
     // Draw multiple trees
    drawTree(10, 28);
     // Draw multiple trees
    drawTree(12, 29);
     // Draw multiple trees
    drawTree(14, 30);

    // Draw multiple trees
    drawTree(14, 27);
     // Draw multiple trees
    drawTree(17, 28);
     // Draw multiple trees
    drawTree(20, 29);
     // Draw multiple trees
    drawTree(23, 30);


    glPushMatrix();

    glTranslatef(-30.0f + playerPosX, 0.6f,30.0f + playerPosZ); // Move the cube
    // Move the player along X-axis

    glColor3f(1.0f, 0.0f, 0.0f); // Set color to red
    glBegin(GL_QUADS);
        // Front face
        glVertex3f(-0.5f, -1.0f, 0.5f);
        glVertex3f(0.5f, -1.0f, 0.5f);
        glVertex3f(0.5f, 1.0f, 0.5f);
        glVertex3f(-0.5f, 1.0f, 0.5f);

        // Back face
        glVertex3f(-0.5f, -1.0f, -0.5f);
        glVertex3f(0.5f, -1.0f, -0.5f);
        glVertex3f(0.5f, 1.0f, -0.5f);
        glVertex3f(-0.5f, 1.0f, -0.5f);

        // Left face
        glVertex3f(-0.5f, -1.0f, -0.5f);
        glVertex3f(-0.5f, -1.0f, 0.5f);
        glVertex3f(-0.5f, 1.0f, 0.5f);
        glVertex3f(-0.5f, 1.0f, -0.5f);

        // Right face
        glVertex3f(0.5f, -1.0f, -0.5f);
        glVertex3f(0.5f, -1.0f, 0.5f);
        glVertex3f(0.5f, 1.0f, 0.5f);
        glVertex3f(0.5f, 1.0f, -0.5f);

        // Top face
        glVertex3f(-0.5f, 1.0f, -0.5f);
        glVertex3f(0.5f, 1.0f, -0.5f);
        glVertex3f(0.5f, 1.0f, 0.5f);
        glVertex3f(-0.5f, 1.0f, 0.5f);

        // Bottom face
        glVertex3f(-0.5f, -1.0f, -0.5f);
        glVertex3f(0.5f, -1.0f, -0.5f);
        glVertex3f(0.5f, -1.0f, 0.5f);
        glVertex3f(-0.5f, -1.0f, 0.5f);
    glEnd();

    glPopMatrix();



    // Update ball physics
    updatePhysics();

    // Reset ball if needed
    resetBallIfNeeded();

    glutSwapBuffers(); // Swap buffers (double buffering)
}

// Function to handle window resizing
void reshape(int w, int h) {
    glViewport(0, 0, w, h);
    glMatrixMode(GL_PROJECTION);
    glLoadIdentity();
    gluPerspective(45.0, (GLfloat)w / (GLfloat)h, 1.0, 100.0);
    glMatrixMode(GL_MODELVIEW);
}

// Timer function to continuously update the scene
void update(int value) {
    glutPostRedisplay();
    glutTimerFunc(16, update, 0); // Update every 16 ms (~60 FPS)
}

int main(int argc, char** argv) {
    // Initialize GLUT
    glutInit(&argc, argv);
    glutInitDisplayMode(GLUT_DOUBLE | GLUT_RGB | GLUT_DEPTH);
    glutInitWindowSize(width, height);
    glutCreateWindow("3D-FP [221010929],[18101434]");

    glEnable(GL_DEPTH_TEST); // Enable depth testing for 3D rendering

    // Set the display, reshape, and key handling functions
    glutDisplayFunc(display);
    glutReshapeFunc(reshape);
    glutKeyboardFunc(keyboard);
    glutKeyboardUpFunc(keyboardUp);
    glutSpecialFunc(specialKeys); // Register the special key callback function (arrow keys)
    glutTimerFunc(25, update, 0); // Set up the timer function to update the scene


    // Start the main loop
    glutMainLoop();

    return 0;
}
