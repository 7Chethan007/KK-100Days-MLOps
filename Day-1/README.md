# Day 1 - Python Virtual Environment Setup for ML Project

## Objective

The xFusionCorp Industries data science team requires a standardized Python environment for a new Machine Learning project.

### Requirements

- Create a Python virtual environment named **ml-env** under `/root/code/`.
- Activate the virtual environment.
- Install the following ML libraries:
  - `numpy`
  - `pandas`
  - `scikit-learn`
  - `matplotlib`
- Generate a `requirements.txt` file using `pip freeze`.
- Save the file at `/root/code/requirements.txt`.

---

## Solution

### Step 1: Verify Working Directory

```bash
root@controlplane ~/code ➜  pwd
```

Output:

```text
/root/code
```

---

### Step 2: Create Virtual Environment

```bash
root@controlplane ~/code ➜  python3 -m venv ml-env
```

This creates a virtual environment named `ml-env` inside `/root/code`.

---

### Step 3: Activate Virtual Environment

```bash
root@controlplane ~/code ➜  source ml-env/bin/activate
```

After activation, the terminal prompt changes and displays the environment name:

```text
(ml-env)
```

---

### Step 4: Upgrade pip

```bash
root@controlplane ~/code via 🐍 v3.12.3 (ml-env) ➜ pip install --upgrade pip
```

Upgrading pip ensures the latest package management features and security fixes.

---

### Step 5: Install Required ML Libraries

```bash
root@controlplane ~/code via 🐍 v3.12.3 (ml-env) ➜ pip install numpy pandas scikit-learn matplotlib
```

Installed packages:

| Package | Purpose |
|----------|----------|
| numpy | Numerical computing |
| pandas | Data manipulation and analysis |
| scikit-learn | Machine learning algorithms |
| matplotlib | Data visualization |

---

### Step 6: Verify Installed Packages

```bash
root@controlplane ~/code via 🐍 v3.12.3 (ml-env) ➜  pip list
```

This displays all installed packages within the virtual environment.

---

### Step 7: Generate requirements.txt

```bash
root@controlplane ~/code via 🐍 v3.12.3 (ml-env) ➜  pip freeze > /root/code/requirements.txt
```

This exports all installed package versions into a requirements file.

---

## Commands Executed

```bash
cd /root/code

python3 -m venv ml-env

source ml-env/bin/activate

pip install --upgrade pip

pip install numpy pandas scikit-learn matplotlib

pip list

pip freeze > /root/code/requirements.txt
```

---

## Key Learning

- Creating isolated Python environments using `venv`.
- Managing dependencies with `pip`.
- Installing Machine Learning libraries.
- Capturing dependencies using `pip freeze`.
- Maintaining reproducible Python environments with `requirements.txt`.

---

## Outcome

Successfully created a Python virtual environment (`ml-env`), installed the required Machine Learning libraries, and generated a `requirements.txt` file for dependency management and reproducibility.

---

**Platform:** KodeKloud Engineer  
**Day:** 1  
**Topic:** Python Virtual Environment & Dependency Management  
