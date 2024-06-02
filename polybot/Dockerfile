# Use an official Python runtime as a parent image
FROM python:3.8

# Set the working directory in the container
WORKDIR /usr/src/app

# Copy the requirements file to the working directory
COPY requirements.txt ./

# Install any needed packages specified in requirements.txt
RUN pip install --no-cache-dir -r requirements.txt

# Copy the rest of the application source code to the working directory
COPY . .

# Make port 8443 available to the world outside this container
EXPOSE 8443

# Define environment variable
ENV TELEGRAM_TOKEN=<your_telegram_token>
ENV TELEGRAM_APP_URL=https://t.me/den1984ber_bot

# Run the app when the container launches
CMD ["python", "app.py"]
