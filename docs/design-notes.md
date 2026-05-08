## 2026-05-07
- XTC Tools 15.3.x on Ubuntu: SetEnv does not reliably add tools to PATH
- Fix: explicitly export the bin directory in ~/.bashrc after sourcing SetEnv
  export PATH=$PATH:~/XMOS/XTC/15.3.0/bin