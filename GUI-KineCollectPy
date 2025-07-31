import cv2
import csv
import time
import pykinect_azure as pykinect
import tkinter as tk
from tkinter import filedialog, scrolledtext, messagebox
from threading import Thread
import os

class KinectApp:
    def __init__(self, root):
        self.root = root
        self.video_file = "output.mkv"
        self.output_file = "skeleton_dataset.csv"
        self.recording = False
        self.running = False
        self.device = None
        self.thread = None
        self.current_label = "Berjalan"
        self.setup_gui()

    def setup_gui(self):
        self.root.title("Kinect Video Recorder and Data Collector")

        frame_top = tk.Frame(self.root)
        frame_top.pack(pady=10, padx=10, anchor="w")

        frame_middle = tk.Frame(self.root)
        frame_middle.pack(pady=10, padx=10, anchor="w")

        frame_bottom = tk.Frame(self.root)
        frame_bottom.pack(pady=10, padx=10, anchor="w")

        tk.Label(frame_top, text="Nama File Video:").grid(row=0, column=0, sticky="w", pady=5)
        self.file_name_entry = tk.Entry(frame_top)
        self.file_name_entry.insert(0, "output.mkv")
        self.file_name_entry.grid(row=0, column=1, pady=5, padx=5)

        tk.Label(frame_top, text="Label Gerakan:").grid(row=1, column=0, sticky="w", pady=5)
        self.label_entry = tk.Entry(frame_top)
        self.label_entry.insert(0, self.current_label)
        self.label_entry.grid(row=1, column=1, pady=5, padx=5)

        tk.Button(frame_middle, text="Pilih File Output Data", command=self.select_output_file).grid(row=0, column=0, sticky="w", pady=5)
        self.output_label = tk.Label(frame_middle, text=f"File Output Data: {self.output_file}")
        self.output_label.grid(row=0, column=1, sticky="w", pady=5, padx=5)

        tk.Button(frame_middle, text="Pilih Folder Video", command=self.select_video_file).grid(row=1, column=0, sticky="w", pady=5)
        self.file_label = tk.Label(frame_middle, text=f"File Video: {self.video_file}")
        self.file_label.grid(row=1, column=1, sticky="w", pady=5, padx=5)

        tk.Button(frame_bottom, text="Mulai Rekam", command=self.start_recording).grid(row=0, column=0, sticky="w", pady=5, padx=5)
        tk.Button(frame_bottom, text="Hentikan Rekam", command=self.stop_recording).grid(row=0, column=1, sticky="w", pady=5, padx=5)

        tk.Label(frame_bottom, text="Data Skeleton:").grid(row=1, column=0, columnspan=2, sticky="w", pady=5)
        self.skeleton_display = scrolledtext.ScrolledText(frame_bottom, height=10, wrap=tk.WORD, state="disabled")
        self.skeleton_display.grid(row=2, column=0, columnspan=2, pady=5, padx=5, sticky="w")

        tk.Button(frame_bottom, text="Keluar", command=self.root.quit).grid(row=3, column=0, sticky="w", pady=10, padx=5)

    def log_to_skeleton_display(self, data):
        self.skeleton_display.configure(state="normal")
        self.skeleton_display.insert(tk.END, f"{data}\n")
        self.skeleton_display.configure(state="disabled")
        self.skeleton_display.see(tk.END)

    def select_output_file(self):
        file_path = filedialog.asksaveasfilename(defaultextension=".csv", filetypes=[("CSV files", "*.csv")])
        if file_path:
            self.output_file = file_path
            self.output_label.config(text=f"File Output Data: {self.output_file}")

    def select_video_file(self):
        folder_path = filedialog.askdirectory()
        if folder_path:
            file_name = self.file_name_entry.get()
            if not file_name.endswith(".mkv"):
                file_name += ".mkv"
            self.video_file = os.path.join(folder_path, file_name).replace("\\", "/")
            self.file_label.config(text=f"File Video: {self.video_file}")

    def start_recording(self):
        if self.recording or self.running:
            return

        self.recording = True
        self.running = True
        self.current_label = self.label_entry.get()
        self.thread = Thread(target=self.record_and_collect_data, daemon=True)
        self.thread.start()

    def stop_recording(self):
        self.recording = False
        self.running = False
        if self.thread and self.thread.is_alive():
            self.thread.join(timeout=2)
        if self.device:
            try:
                self.device.stop_cameras()
                self.device = None
            except Exception as e:
                print(f"Error stopping device: {e}")
        try:
            cv2.destroyWindow('Color Image')
        except:
            pass

    def record_and_collect_data(self):
        joint_names = ["Pelvis", "SpineNaval", "SpineChest", "Neck", "ClavicleLeft", "ShoulderLeft", "ElbowLeft", "WristLeft", "HandLeft", "HandTipLeft", "ThumbLeft", "ClavicleRight", "ShoulderRight", "ElbowRight", "WristRight", "HandRight", "HandTipRight", "ThumbRight", "HipLeft", "KneeLeft", "AnkleLeft", "FootLeft", "HipRight", "KneeRight", "AnkleRight", "FootRight", "Head", "Nose", "EyeLeft", "EarLeft", "EyeRight", "EarRight"]
        header = ["timestamp", "frame_id"] + [f"{name}_{axis}" for name in joint_names for axis in ["x", "y", "z"]] + ["label"]
        pykinect.initialize_libraries(track_body=True)

        device_config = pykinect.default_configuration
        device_config.color_format = pykinect.K4A_IMAGE_FORMAT_COLOR_BGRA32
        device_config.color_resolution = pykinect.K4A_COLOR_RESOLUTION_1080P
        device_config.depth_mode = pykinect.K4A_DEPTH_MODE_WFOV_2X2BINNED

        self.device = pykinect.start_device(config=device_config, record=True, record_filepath=self.video_file)
        bodyTracker = pykinect.start_body_tracker()

        with open(self.output_file, mode="w", newline="") as file:
            writer = csv.writer(file)
            writer.writerow(header)

            cv2.namedWindow('Color Image', cv2.WINDOW_NORMAL)
            frame_id = 0

            try:
                while self.running:
                    capture = self.device.update()
                    body_frame = bodyTracker.update()
                    ret_color, color_image = capture.get_color_image()
                    if not ret_color:
                        continue

                    if body_frame:
                        color_image = body_frame.draw_bodies(color_image, pykinect.K4A_CALIBRATION_TYPE_COLOR)
                        bodies = body_frame.get_bodies()
                        if bodies:
                            for body in bodies:
                                joints = []
                                display_text = f"Frame {frame_id}\n"
                                for joint_index, joint in enumerate(body.joints):
                                    joint_names = ["Pelvis", "SpineNaval", "SpineChest", "Neck", "ClavicleLeft", "ShoulderLeft", "ElbowLeft", "WristLeft", "HandLeft", "HandTipLeft", "ThumbLeft", "ClavicleRight", "ShoulderRight", "ElbowRight", "WristRight", "HandRight", "HandTipRight", "ThumbRight", "HipLeft", "KneeLeft", "AnkleLeft", "FootLeft", "HipRight", "KneeRight", "AnkleRight", "FootRight", "Head", "Nose", "EyeLeft", "EarLeft", "EyeRight", "EarRight"]
                                    joint_name = joint_names[joint_index] if joint_index < len(joint_names) else f"Joint {joint_index}"
                                    if joint.confidence_level >= 2:
                                        joints.extend([joint.position.x, joint.position.y, joint.position.z])
                                        display_text += f"{joint_name}: ({joint.position.x:.2f}, {joint.position.y:.2f}, {joint.position.z:.2f})\n"
                                    else:
                                        joints.extend([None, None, None])
                                        display_text += f"{joint_name}: (Low Confidence)\n"
                                timestamp = time.time()
                                row = [timestamp, frame_id] + joints + [self.current_label]
                                writer.writerow(row)
                                self.log_to_skeleton_display(display_text.strip())
                            frame_id += 1
                    cv2.imshow('Color Image', color_image)
                    if cv2.waitKey(1) == ord('q') or not self.running:
                        break

            except Exception as e:
                print(f"Error: {e}")
            finally:
                if self.device:
                    try:
                        self.device.stop_cameras()
                        self.device = None
                    except Exception as e:
                        print(f"Error during final device stop: {e}")
                cv2.destroyWindow('Color Image')
                self.recording = False
                self.log_to_skeleton_display("Data skeleton sudah tersimpan.")
                self.log_to_skeleton_display(f"Video rekaman telah tersimpan di: {self.video_file}")
                messagebox.showinfo("Selesai", f"Data disimpan di:{self.output_file} Video di:{self.video_file}")

if __name__ == "__main__":
    root = tk.Tk()
    app = KinectApp(root)
    root.mainloop()
