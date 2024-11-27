# Auto-Scaling-EC2-using-Terra



# AWS Provider Configuration
provider "aws" {
  region = "us-east-1" # Replace with your desired AWS region
}

# Security Group to allow HTTP and SSH traffic
resource "aws_security_group" "web_sg" {
  name        = "web_sg"
  description = "Allow HTTP and SSH traffic"

  ingress {
    description = "Allow HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "Allow SSH"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Launch Template for Auto Scaling Group
resource "aws_launch_template" "web_launch_template" {
  name_prefix   = "web_launch_template"

  image_id      = "ami-0915bcb5fa77e4892" # Amazon Linux 2
  instance_type = "t2.micro"
  key_name      = "auto-scaling" # Replace with your EC2 key pair name

  network_interfaces {
    associate_public_ip_address = true
    security_groups             = [aws_security_group.web_sg.id]  # Use security group ID here
  }

  user_data = base64encode(<<-EOF
    #!/bin/bash
    yum update -y
    yum install -y httpd
    echo "Hello, this is a blog about MS Dhoni!" > /var/www/html/index.html
    systemctl start httpd
    systemctl enable httpd
  EOF
  )
}

# Auto Scaling Group (Single Block)
resource "aws_autoscaling_group" "web_asg" {
  desired_capacity          = 1
  min_size                  = 1
  max_size                  = 3
  vpc_zone_identifier       = ["subnet-0d9409ea9003cd699", "subnet-0a68edd0df11eee9e"]  # Updated with two subnets
  health_check_type         = "EC2"
  health_check_grace_period = 300

  launch_template {
    id      = aws_launch_template.web_launch_template.id
    version = "$Latest"
  }
  tag {
    key                 = "Name"
    value               = "blog_instance"
    propagate_at_launch = true
  }
}
  
# Load Balancer (Needs two subnets in different AZs)
resource "aws_lb" "web_lb" {
  name               = "web-lb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.web_sg.id]
  subnets            = [
    "subnet-0d9409ea9003cd699",  # Subnet in us-east-1a
    "subnet-0a68edd0df11eee9e"   # Subnet in us-east-1c
  ]
}
     
# Target Group
resource "aws_lb_target_group" "web_tg" {
  name        = "web-tg"
  port        = 80
  protocol    = "HTTP"
  vpc_id      = "vpc-03b42b7942a0e71b5" # Replace with your actual VPC ID
}
  
# Listener
resource "aws_lb_listener" "web_listener" {
  load_balancer_arn = aws_lb.web_lb.arn
  port              = 80
  protocol          = "HTTP"
    
  default_action {
    type             = "forward"

target_group_arn = aws_lb_target_group.web_tg.arn
  }
}
    
# CloudWatch Alarm for CPU Utilization
resource "aws_cloudwatch_metric_alarm" "cpu_high" {
  alarm_name          = "HighCPUUtilization"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = 300 
  statistic           = "Average"   
  threshold           = 80
  
  dimensions = {
    AutoScalingGroupName = aws_autoscaling_group.web_asg.name
  }
 
  alarm_actions = [aws_autoscaling_policy.scale_out.arn]
}

# Auto Scaling Policy   
resource "aws_autoscaling_policy" "scale_out" {
  name                   = "scale-out"
  scaling_adjustment     = 1
  adjustment_type        = "ChangeInCapacity"
  autoscaling_group_name  = aws_autoscaling_group.web_asg.name
}![image](https://github.com/user-attachments/assets/f9094205-72ee-4ae0-a180-ab4fa066e0b1)
