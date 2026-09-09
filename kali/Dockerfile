FROM kalilinux/kali-rolling
RUN apt-get update && apt-get upgrade -y
RUN apt-get install nmap -y
RUN apt-get install sudo -y
RUN apt-get install net-tools -y
RUN useradd -m -s /bin/bash -p '$6$cyD6xRoLvRrbebUn$Hxn.9IX4QzXiuek2ONikMbtYnurr8ClX4cfVWrsGem97dbedOH33G6wmQSH1cpXKQNYqbJZBP8iJmA19f6d/70' attacker
RUN usermod -aG sudo attacker
RUN echo 'root:$6$hBQjIhUJPAcaE8rH$pYjrJmks1PgvApxI5AJZjhh8TwAa7S5rLVB7Y0x.XNxwKR4poAxWRU6snslpBiURfSRljWfebZz3eF5fB881x0'| chpasswd -e
CMD ["su", "attacker"]
