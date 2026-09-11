FROM kalilinux/kali-rolling
RUN apt-get update && apt-get upgrade -y
RUN apt-get update && apt-get install -y --no-install-recommends \
    nmap \
    sqlmap \
    iputils-ping \
    sudo \
    net-tools \
    && rm -rf /var/lib/apt/lists/*
RUN useradd -m -s /bin/bash -p '$6$cyD6xRoLvRrbebUn$Hxn.9IX4QzXiuek2ONikMbtYnurr8ClX4cfVWrsGem97dbedOH33G6wmQSH1cpXKQNYqbJZBP8iJmA19f6d/70' attacker
RUN usermod -aG sudo attacker
RUN echo 'root:$6$hBQjIhUJPAcaE8rH$pYjrJmks1PgvApxI5AJZjhh8TwAa7S5rLVB7Y0x.XNxwKR4poAxWRU6snslpBiURfSRljWfebZz3eF5fB881x0'| chpasswd -e
CMD ["su", "attacker"]
