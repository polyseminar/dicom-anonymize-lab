# DICOM anonymize lab

Anonymize a DICOM study with the Basic profile from the DICOM standard, check every tag that changed, and then remove the patient details that the profile leaves burned into the pixels, on your own computer, for learning. The lab runs an [Orthanc](https://www.orthanc-server.com/) PACS with its pixels masker plugin, and [DCMTK](https://dcmtk.org), the OFFIS DICOM command-line tools.

This lab goes with the Polyseminar video "Anonymize DICOM images: tags and burned-in text". It is the fourth lab in the series, after [`pacs-lab`](https://github.com/polyseminar/pacs-lab), [`dicom-send-lab`](https://github.com/polyseminar/dicom-send-lab) and [`dicom-errors-lab`](https://github.com/polyseminar/dicom-errors-lab). This one stands alone, so you don't need the others.

The [lab page](https://polyseminar.com/labs/anonymize-a-dicom-study) on polyseminar.com has the video, its chapters, and the main commands.

> **Lab use only.** Load only the public test data named below. Never load real patient data, including your own scans, and never point these tools at a hospital system without permission. This lab teaches the method; it is not a validated de-identification tool, and this README is not legal advice.

## What you need

- **Docker**, with at least 8 GB of RAM in the computer:

  | System | What to install |
  | :--- | :--- |
  | Windows 11 | Docker Desktop, then WSL. See [Install Docker on Windows](#install-docker-on-windows) below. |
  | macOS | [Docker Desktop](https://docs.docker.com/desktop/setup/install/mac-install/) for Apple silicon or Intel |
  | Linux | [Docker Engine](https://docs.docker.com/engine/install/) with the Compose plugin |

- **About 3.5 GB of free disk space.** The Orthanc image uses about 2.6 GB, the tools helper about 250 MB, the downloaded study about 220 MB, and the PACS about 250 MB once it holds the study and its anonymized copy.

### Install Docker on Windows

1. Open the **Microsoft Store**, search for **Docker Desktop**, and click **Install**. It does not ask for an administrator.
2. Turn on WSL, the Windows Subsystem for Linux, which Docker Desktop runs on. Click Start, type `Terminal`, right-click **Terminal**, and choose **Run as administrator**. Then run:

   ```
   wsl --install --no-distribution
   ```

3. Restart the computer.
4. Start Docker Desktop and wait until it shows **Engine running**.

Docker Desktop is free for personal use, education, and small businesses (fewer than 250 employees and less than $10 million in annual revenue). Larger organizations and government entities need a paid subscription ([Docker Desktop license terms](https://docs.docker.com/subscription-billing/desktop-license/)). On a work computer, ask your IT team before you install it.

## Get the lab

Download **[dicom-anonymize-lab.zip](https://github.com/polyseminar/dicom-anonymize-lab/releases/latest/download/dicom-anonymize-lab.zip)** and extract it. You get one folder, `dicom-anonymize-lab`, with `compose.yaml` inside. Open a terminal in that folder:

| System | How |
| :--- | :--- |
| Windows 11 | In File Explorer, right-click the ZIP and choose **Extract All**, then **Extract**. Right-click the `dicom-anonymize-lab` folder and choose **Open in Terminal** |
| macOS | Double-click the ZIP. Open Terminal, type `cd` and a space, drag the folder into the window, and press Enter |
| Linux | Extract the ZIP in the file manager, right-click the folder, and choose **Open in Terminal** |

You can also clone the repository. GitHub's own **Code > Download ZIP** works too, but on Windows it puts the folder inside a folder of the same name; use the inner one.

If a lab from an earlier video is running, stop it first with `docker compose down` in its folder: all four labs use ports 8042 and 4242.

## Start the lab

The commands are the same in PowerShell, Command Prompt, macOS Terminal, and Linux shells. Docker Desktop must be running first.

1. Start the PACS:

   ```
   docker compose up -d
   ```

2. Download the test study into Docker:

   ```
   docker compose run --rm download
   ```

   It ends with a list of folders: one collection, one patient, two studies, and one series in each study.

3. Send the newer study to the PACS, with a screen capture:

   ```
   docker compose run --rm tools load
   ```

   The first run builds the tools helper. `load` makes a screen capture from one CT slice, with the patient's name, ID and study date written into the image, the way some workstations label one. Then it sends the study's 226 CT images and the capture to the PACS. It ends with `Sent 226 CT images and 1 screen capture to the PACS`.

4. Open <http://localhost:8042> and sign in with user `orthanc` and password `orthanc`. The list shows one study, patient `PAN_03`.

## Anonymize the study

In the browser, click the study to open it, and click **Anonymize**, the button with a person on it. Keep the new name and ID that Orthanc fills in.

Clear the **StudyDescription** field. The dialog fills it in from the original, and whatever it holds goes into the copy, although the profile removes that tag. Delete the text; the trash icon next to the field doesn't help, because in Orthanc Explorer 2 1.15.0 the dialog still sends the value.

Click **Anonymize**, wait for the bar to fill, click **Close**, and refresh the page. Orthanc applies the Basic Application Level Confidentiality Profile from DICOM PS3.15, Annex E, writes the result as a new study, and keeps the original. The list now shows two studies.

The profile gives each attribute an action from its Table E.1-1:

| Action | What it does |
| :--- | :--- |
| D | Replace the value with a dummy |
| Z | Empty the value, or replace it with a dummy |
| X | Remove the attribute |
| K | Keep it |
| C | Clean it: replace identifying values with similar ones that don't identify |
| U | Replace a UID with a new one, the same new one wherever it appears |

Options in PS3.15, E.3, keep more (for example dates, for research that follows a patient over time) or clean more (for example text in the pixels).

## Check what changed

```
docker compose run --rm tools compare
```

`compare` reads CT image 111 and the screen capture from the original study and from the anonymized copy, and lists what happened to each tag:

```
CT image 111: 215 tags before, 78 after
  Replaced  PatientName                        PAN_03                                       → Anonymized 1a52a67e
  Replaced  PatientID                          PAN_03                                       → 1a52a67e-ddab-42b9-919f-55179722b724
  Replaced  DeidentificationMethod             Per DICOM PS 3.15 AnnexE. Details in 0012,0… → Orthanc 1.13.0
  Replaced  5 UIDs: SOPInstanceUID, IrradiationEventUID, StudyInstanceUID, SeriesInstanceUID, FrameOfReferenceUID
  Emptied   StudyDate, ContentDate, StudyTime, ContentTime, StudyDescription, PatientSex
  Removed   PatientAge                         025Y
  Removed   SeriesDescription                  1.25mm Pancreas A
  ...
  Removed   131 more, 122 of them private tags

Screen capture: 30 tags before, 32 after
  ...
  Added     PatientIdentityRemoved             YES
  Added     DeidentificationMethod             Orthanc 1.13.0
  Kept      BurnedInAnnotation                 YES
```

Your new name, ID and UIDs differ: the dialog makes a random ID each time, and the name starts with the ID's first eight characters.

Read both de-identification tags. TCIA had already de-identified this data set, and its De-identification Method Code Sequence (0012,0064) lists the options it used. Orthanc replaces the De-identification Method text (0012,0063) but keeps that sequence, so the copy still says "Retain Longitudinal Temporal Information Modified Dates", although Orthanc emptied the dates. The text itself says only `Orthanc 1.13.0`, because the dialog sends its own values for some tags. A plain `POST /studies/{id}/anonymize` with no replacements names the profile: `Orthanc 1.13.0 - PS 3.15-2023b Table E.1-1 Basic Profile`.

## Text in the pixels

Open the anonymized study, then its OT series, the screen capture, and its preview. The header is clean, but the image still shows `PAN_03 PAN_03 20040110` across the top. The profile changes tags; it doesn't look at pixels. Burned In Annotation (0028,0301) still says `YES`.

Text in the pixels is the Clean Pixel Data option (PS3.15, E.3.1): remove it, by any means, and set Burned In Annotation to `NO`. Screen captures, scanned documents and ultrasound images often carry text like this.

## Mask it

The label fits in a box from the top left corner, 320 pixels wide and 40 high:

```
docker compose run --rm tools mask 0 0 320 40
```

`mask` takes the box as `x0 y0 x1 y1`, in pixels from the top left corner. It uses Orthanc's pixels masker plugin to fill the box with black in the anonymized study's screen capture, sets Burned In Annotation to `NO`, and replaces the unmasked capture. The plugin doesn't find text itself: you give it the box. Refresh the browser and open the capture again: the text is gone. Run `compare` again, and the capture shows `Replaced BurnedInAnnotation YES → NO`.

## Before you share a study

1. **The profile:** the Basic one, plus the options this recipient needs.
2. **The pixels:** every image that might carry text.
3. **Private tags:** removed, unless you've checked what's in them.
4. **The link back:** this PACS keeps the original study next to the copy, so whoever runs it can match the two. Under the GDPR, data that can be linked back is pseudonymised, and still personal data.

## Stop and reset

| Command | Result |
| :--- | :--- |
| `docker compose down` | Stops the PACS. Your studies stay. |
| `docker compose down -v` | Stops the PACS and deletes the studies stored in it. The downloaded study stays; run `load` again to start over. |
| `docker compose --profile tools down -v --rmi local` | Deletes everything: the stored studies, the downloaded study, `/work`, and the tools helper. |

## What runs

| Part | Detail |
| :--- | :--- |
| `pacs` | `orthancteam/orthanc:26.9.1` (Orthanc 1.13.0) with Orthanc Explorer 2, the OHIF viewer, DICOMweb, and the pixels masker plugin (0.1.4). Web interface on `localhost:8042`, DICOM on `localhost:4242`, AE title `ORTHANC`. It accepts images from any sender |
| `download` helper | Runs the NCI Imaging Data Commons tool `idc-index` 0.12.5 in a temporary container and saves the study in the `study` volume |
| `tools` helper | DCMTK 3.6.9 and Python from Debian 13, built on first use, with the lab's scripts: `make-capture`, `load`, `compare` and `mask`. Reads the study read-only, writes to the `work` volume, and reaches the PACS as `pacs` (DICOM on 4242, the REST API on 8042) |

Both ports are bound to `127.0.0.1`, so only this computer can reach them. The `orthanc` password is acceptable only for that reason. Do not expose this lab to a network.

`mask` doesn't use the plugin's `KeepSource: false`: in plugin 0.1.4 that deletes the source series and then fails, so nothing is stored. It deletes the unmasked series itself, after the masked one is stored.

## Test data

The lab uses patient `PAN_03` from the CTpred-Sunitinib-panNET collection: two contrast-enhanced abdominal CT studies with 422 images in total, from a GE Discovery CT750 HD scanner with 1.25 mm slices. `load` sends the newer study (226 images). `make-capture` saves its slice 111 as a picture in a new Secondary Capture object and writes the data set's own codes on it (`PAN_03` is TCIA's code, not a name, and the dates were shifted before publication).

**Citation:**
Chen, L., Wang, W., Jin, K., Yuan, B., Tan, H., Sun, J., Guo, Y., Luo, Y., Feng, S.-ting, Yu, X., Chen, M.-hu, & Chen, J. (2022). *Prediction of Sunitinib Efficacy using Computed Tomography in Patients with Pancreatic Neuroendocrine Tumors (CTpred-Sunitinib-panNET)* (Version 1) [Data set]. The Cancer Imaging Archive. <https://doi.org/10.7937/SPGK-0P94>

- **License:** [Creative Commons Attribution 4.0 (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/). The lab downloads the files unchanged; the Secondary Capture is an adaptation of one image.
- **Data use:** follow the [TCIA Data Usage Policy](https://www.cancerimagingarchive.net/data-usage-policies-and-restrictions/). Credit the dataset whenever you show it, and do not try to identify or contact the people in it.
- **Download service:** NCI Imaging Data Commons. Fedorov, A., et al. (2023). National Cancer Institute Imaging Data Commons: Toward Transparency, Reproducibility, and Scalability in Imaging Artificial Intelligence. *RadioGraphics*, 43(12). <https://doi.org/10.1148/rg.230180>

## Sources

- DICOM PS3.15, Annex E: Table E.1-1 (Basic Application Level Confidentiality Profile) and E.3.1 (Clean Pixel Data Option). <https://www.dicomstandard.org/current>
- Orthanc anonymization: <https://orthanc.uclouvain.be/book/users/anonymization.html>
- Orthanc pixels masker plugin: <https://orthanc.uclouvain.be/book/plugins/pixels-masker.html>
- GDPR Article 4(5) and Recital 26 (pseudonymisation)

## Troubleshooting

| Problem | Fix |
| :--- | :--- |
| `docker` is not recognized, or `command not found` | Install Docker, then open a new terminal. |
| `no configuration file provided: not found` | The terminal is not in the lab folder. Open it in the folder that contains `compose.yaml`. |
| `Cannot connect to the Docker daemon`, or an error that mentions `dockerDesktopLinuxEngine` | Start Docker Desktop and wait until it shows that the engine is running. On Linux, start the Docker service. |
| `permission denied` on Linux | Put `sudo` in front of the command, or add your user to the `docker` group. |
| `port is already allocated` | Another program uses port 8042 or 4242, often the PACS from an earlier lab. Run `docker compose down` in that lab's folder, or stop the other program. |
| `load` fails at once, before `Sent` | Download the study first (Start the lab, step 2). |
| `compare` or `mask` says `No anonymized study yet` | Anonymize the study in the browser first. |
| `compare` shows the old study after a second anonymization | It compares the newest anonymized copy. Delete older copies in the browser, or reset with `docker compose down -v` and `load`. |
| The sign-in prompt keeps returning | Use user `orthanc` and password `orthanc`. |
| The download stops with a network error | Run it again. |

## License

The lab files (`compose.yaml`, this README) are under the [MIT License](LICENSE). The software the lab runs keeps its own license: Orthanc is GPL, DCMTK is BSD-style, OHIF is MIT, and the test data is CC BY 4.0 (see Test data). The Polyseminar videos and course materials are not covered by this license; they are © Polyseminar, all rights reserved.

## Independence

Polyseminar is an independent education publisher. It is not affiliated with or endorsed by a certification body, professional society, healthcare provider, or imaging vendor. DCMTK, Orthanc, OHIF, TCIA, and IDC are the work of their own authors.
